
# Chaos experiments

A simple ansible playbook to run chaos experiments on infrastructure that have been deployed via [this project](https://github.com/fishaffair/sre-cource-deploy).

Playbook choses a random host for every group in inventory:

 - etcd_cluster
 - postgres_cluster (patroni and db)
 - haproxy (balancer)

And runs several experiments:

- Network packet loss / delay
- RAM / CPU / IO load
- Service stop

> [!NOTE]
> (you can disable some of it via ``vars.yml``)

After specified delay the rollback role begin put everything in its place.

### Experiment specification

1. Firstly, clone repo and check playbook on your hosts:

`ansible-playbook chaos.yml -i inventory --check`

 and run it:

`ansible-playbook chaos.yml -i inventory`

2. After launching the experiment, we expect to receive notifications about etcd lost.  During chaos test I pushing the small load to ifarustucture via [my k6 tests](https://github.com/fishaffair/k6-test).

In the etcd and patroni log files, there wiil be a message about the change of etcd leader and patroni master.

The cluster will be active only then at least:
- one patroni 
- two etcd 
is available.

3. See system's reaction (check alerts and logs on etcd or patroni cluster)

As example i have email alerts from prometheus-aletmanager that notify me about cluster.

<details>
  <summary>Screnshots and logs</summary>
  
  Logs from etcd:
 
```
e1f06668267121f5 [term 38] received MsgTimeoutNow from b586ded327f9460d and starts an election to get leadership."
e1f06668267121f5 lost leader b586ded327f9460d at term 39"
e1f06668267121f5 became leader at term 39"
1f06668267121f5 elected leader e1f06668267121f5 at term 39"
```
  
Logs from patroni:

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
  
  Email alert:
 
  ![Alt text](/img/etcd_alert.png "etcd_alert")
  
  ![Alt text](/img/cpu_alert.png "cpu_alerts")
  
  Grafana:

  ![Alt text](/img/patroni_grafana.png "patroni_grafana")
  
  ![Alt text](/img/etcd_grafana.png "etcd_grafana")
	

</details>


4. Result analysis:

After the end of the experiment, everything returned as it was before the script was launched, services on the required hosts were started, and the cluster was restored to its original state with exchanged patroni master.

<details>
  <summary>Cluster up</summary>
  
etcd:

```
+------------------+---------+--------+-----------------------+-----------------------+------------+
|        ID        | STATUS  |  NAME  |      PEER ADDRS       |     CLIENT ADDRS      | IS LEARNER |
+------------------+---------+--------+-----------------------+-----------------------+------------+
| b586ded327f9460d | started | etcd-2 | http://10.0.10.6:2380 | http://10.0.10.6:2379 |      false |
| e1f06668267121f5 | started | etcd-3 | http://10.0.10.7:2380 | http://10.0.10.7:2379 |      false |
| e8cc3f7ff72fe07d | started | etcd-1 | http://10.0.10.5:2380 | http://10.0.10.5:2379 |      false |
+------------------+---------+--------+-----------------------+-----------------------+------------+
```

patroni:

```
+ Cluster: postgres-cluster ---+-----------+----+-----------+
| Member | Host      | Role    | State     | TL | Lag in MB |
+--------+-----------+---------+-----------+----+-----------+
| psql-1 | 10.0.10.3 | Replica | streaming | 47 |         0 |
| psql-2 | 10.0.10.4 | Leader  | running   | 47 |           |
+--------+-----------+---------+-----------+----+-----------+
```

</details>

Notifications related to the operating system such as: disk, memory, and RAM load the CPU most of the time,  receives a notification about the CPU load, although alerts are configured for other types. The solution would be to lower the thresholds for other types of attacks (disk, memory) in order to receive them earlier.
