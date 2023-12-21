
# Chaos experiments

A simple ansible playbook to run chaos experiments on infrastructure that have been deployed via [this project](https://github.com/fishaffair/sre-cource-deploy).

Playbook choses a random host for every group in inventory:

 - etcd_cluster
 - postgres_cluster (patroni and psql)
 - haproxy (balancer)

And runs several experiments:

- Network packet loss / delay
- RAM / CPU / IO load
- Service stop (etcd,patroni)

> [!NOTE]
> you can disable some of it in ``vars.yml``

After specified delay the rollback role begins to put everything in its place.

### How to run experiments?

Firstly, clone repo and check playbook on your hosts:

`ansible-playbook chaos.yml -i inventory --check`

 and run it:

`ansible-playbook chaos.yml -i inventory`

> [!WARNING]
> The cluster will be active only then at least:
> - one patroni
> - two etcd
> is available.

### Main experiments:

## 1. Patroni failover
  **1.1 Experiment description:** stop patroni service on current patroni leader with [systemd role](roles/systemd/tasks/main.yml) .

  **1.2 Expected results:** patroni replica has promoted self to a new leader

  **1.3 Real outcomes:**
  
  Patroni logs:
  
```
INFO: Cleaning up failover key after acquiring leader lock...
INFO:patroni.watchdog.base:Software Watchdog activated with 25 second timeout, timing slack 15 seco>
INFO: Software Watchdog activated with 25 second timeout, timing slack 15 s>
INFO:patroni.__main__:promoted self to leader by acquiring session lock INFO:patroni.ha:Lock owner: psql-2; I am psql-2
INFO: promoted self to leader by acquiring session lock
INFO:patroni.__main__:updated leader lock during promote
INFO: Lock owner: psql-2; I am psql-2
INFO: updated leader lock during promote
```
  
![Alt text](/img/patroni_grafana.png "patroni_grafana")  

  1.4 **Results analysis:** patroni replica has promoted self to a new leader as espected
  
## 2. Etcd failover
  **2.1 Experiment description:** stop etcd service on current etcd leader with [systemd role](roles/systemd/tasks/main.yml) .

  **2.2 Expected results:** etcd hosts have a valid quorum with at least two active nodes, so new leader has reelected accordingly

  **2.3 Real outcomes:** etcd hosts have a valid quorum with at least two active nodes, so new leader has reelected accordingly
  
   ![Alt text](/img/etcd_alert.png "etcd_alert")
   ![Alt text](/img/etcd_grafana.png "etcd_grafana")

```
e1f06668267121f5 [term 38] received MsgTimeoutNow from b586ded327f9460d and starts an election to get leadership."
e1f06668267121f5 lost leader b586ded327f9460d at term 39"
e1f06668267121f5 became leader at term 39"
1f06668267121f5 elected leader e1f06668267121f5 at term 39"
``` 
  **2.4 Results analysis:** during quorum desigion has elected new leader 

## 3. Network delay
  **3.1 Experiment description:** create network packet loss via [tc](roles/network/tasks/main.yml) to patroni master

  **3.2 Expected results:** increased latency from blackbox to any API request

  **3.3 Real outcomes:** latency has increased:
  
  ![Alt text](/img/blackbox_probe_grafana.png "blackbox_probe_grafana")
  ![Alt text](/img/blackbox_alert.png "blackbox_alert")

  **3.4 Results analysis:** negative impact on request latency

## 4. Cpu load
  **4.1 Experiment description:** push maximum load on CPU with [stress-ng](roles/os/tasks/main.yml)

  **4.2 Expected results:** email alert from alertmanager and latency bump

  **4.3 Real outcomes:** CPU stress test did not show a significant impact on latency for API requests
  
   ![Alt text](/img/cpu_alert.png "cpu_alert")

  **4.4 Results analysis:** cpu load on patroni master 

## 5. Alerting test
  **5.1 Experiment description:** check alertmanager

  **5.2 Expected results:** new email alerts from prometheus alertmanager

  **5.3 Real outcomes:** Have email alerts due to abnormal cluster state
  
  ![Alt text](/img/etcd_alert.png "etcd_alert")
  ![Alt text](/img/blackbox_alert.png "blackbox_alert")

**5.4 Results analysis:** Notifications related to the operating system such as: disk, memory, and RAM load the CPU most of the time,  receives a notification about the CPU load, although alerts are configured for other types. The solution would be to lower the thresholds for other types of attacks (disk, memory) in order to receive them earlier.