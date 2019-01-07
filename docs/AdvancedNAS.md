# Tutorial for Advanced Neural Architecture Search
Currently many of the NAS algorithms leverage the technique of **weight sharing** among trials to accelerate its training process. For example, [ENAS][1] delivers 1000x effiency with '_parameter sharing between child models_', compared with the previous [NASNet][2] algorithm. Other NAS algorithms such as [DARTS][3], [Network Morphism][4], and [Evolution][5] is also leveraging, or has the potential to leverage weight sharing.

This is a tutorial on how to enable weight sharing in NNI. 

## Weight Sharing among trials
Currently we recommend sharing weights through NFS (Network File System), which supports sharing files across machines, and is light-weighted, (relatively) efficient. We also welcome contributions from the community on more efficient techniques.

### NFS Setup
In NFS, files are physically stored on a server machine, and trials on the client machine can read/write those files in the same way that they access local files.

#### Install NFS on server machine
First, install NFS server:
```bash
sudo apt-get install nfs-kernel-server
```
Suppose `/tmp/nni/shared` is used as the physical storage, then run:
```bash
sudo mkdir -p /tmp/nni/shared
sudo echo "/tmp/nni/shared *(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
sudo service nfs-kernel-server restart
```
You can check if the above directory is successfully exported by NFS using `sudo showmount -e localhost`

#### Install NFS on client machine
First, install NFS client:
```bash
sudo apt-get install nfs-common
```
Then create & mount the mounted directory of shared files:
```bash
sudo mkdir -p /mnt/nfs/nni/
sudo mount -t nfs 10.10.10.10:/tmp/nni/shared /mnt/nfs/nni
```
where `10.10.10.10` should be replaced by the real IP of NFS server machine in practice.

### Weight Sharing through NFS file
With the NFS setup, trial code can share model weight through loading & saving files. For example, in tensorflow:
```python
# save models
saver = tf.train.Saver()
saver.save(sess, os.path.join(params['save_path'], 'model.ckpt'))
# load models
tf.init_from_checkpoint(params['restore_path'])
```
where `'save_path'` and `'restore_path'` in hyper-parameter can be managed by the tuner.

## Asynchornous Dispatcher Mode for trial dependency control
The feature of weight sharing enables trials from different machines, in which most of the time **read after write** consistency must be assured. After all, the child model should not load parent model before parent trial finishes training. To deal with this, users can enable **asynchronous dispatcher mode** with `multiThread: true` in `config.yml` in NNI, where the dispatcher assign a tuner thread each time a `NEW_TRIAL` request comes in, and the tuner thread can decide when to submit a new trial by blocking and unblocking the thread itself. For example:
```python
    def generate_parameters(self, parameter_id):
        self.thread_lock.acquire()
        indiv = # configuration for a new trial
        self.events[parameter_id] = threading.Event()
        self.thread_lock.release()
        if indiv.parent_id is not None:
            self.events[indiv.parent_id].wait()

    def receive_trial_result(self, parameter_id, parameters, reward):
        self.thread_lock.acquire()
        # code for processing trial results
        self.thread_lock.release()
        self.events[parameter_id].set()
```


[1]: https://arxiv.org/abs/1802.03268
[2]: https://arxiv.org/abs/1707.07012
[3]: https://arxiv.org/abs/1806.09055
[4]: https://arxiv.org/abs/1806.10282
[5]: https://arxiv.org/abs/1703.01041 