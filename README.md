# Neuroevolution

Replication of [Uber AI Labs Neuroevolution paper](https://arxiv.org/pdf/1712.06567.pdf).

[![Frostbite Agent](img/frostbite.png)](https://www.youtube.com/watch?v=rotoBxjUBmM)

> Graph of training curve goes here

## ToDo

- Run on a second environment
- Training graph in readme
- Video in readme
- Find a nice way to store and show found policy seeds
- Polish and finish


## Approach

I use a master-worker architecture, with:

- Golang master
    - gRPC server that controls the genetic algorithm internals
    - Sends sequences of seeds (individuals) to workers
- Python workers
    - Each has a copy of the policy network and environment
    - Swaps out full network weights on each evaluation (without rebuilding network)
    - Sends seeds and their scores back to master

This results in a simpler system than Uber / OpenAIs approach, which uses redis as an intermediary,
and requires workers to synchronise.


## Deployment

### Building Images

Both master and worker are packaged as docker containers. Either pull the containers from docker-hub,
or build them yourself:
```
# Download
docker pull cshenton/neuro:worker
docker pull cshenton/neuro:master

# Build
docker build -t cshenton/neuro:worker -f worker/Dockerfile .
docker build -t cshenton/neuro:master -f master/Dockerfile .
```

### Launching Cluster

Cloudformation scripts deploy the experiment. The following information is required:
- Availability Zone
- VPC
- Gym Environment Name
- Number of workers

Then the cloudformation scripts create:
- Master
    - Security group (Open on 8080)
    - On-demand instance (`c5.large`)
    - ECS Task
    - Single container ECS Service
    - Log group
- Workers
    - Security group (no ingress)
    - Spot fleet of desired size (`c5.18xlarge`)
    - ECS Task (1 vCPU per container)
    - ECS Service with `numWorkers` tasks
    - Log group

Unfortunately, AWS limits spot fleets to 360 vCPUs per account by default, which is 180 discrete
cpu cores. Therefore, we run 180 workers by default, and larger attempts will fail to fulfill
the bigger spot request.

Running this 180 worker fleet for an hour costs at most `0.04*180 + 0.1 = $7.3`, with more likely
prices hovering around `$4.80`.

For comparison, a p3 instance in the same region (`ap-southeast-2`) costs `$12.24` per hour for a
four NVIDIA Tesla V100 GPU instance.


## Discussion

I'd recommend keeping a slightly larger truncated population (50, instead of 10 in the paper). This
keeps the population robust to domination by individuals who had lucky runs, and achieved evaluation
scores high above their average.

Otherwise, the vanilla GA with a single run tends to reward policies with high variance. Maintaining
a larger truncated population is a cheaper solution to this than doing multiple evaluation runs per
candidate policy.



## Protobufs

`gRPC` is used to enable simple communication between workers and master. Server, client stubs
are generated as follows.

```
# python
python -m grpc_tools.protoc -I . proto/neuroevolution.proto --python_out=. --grpc_python_out=.

# golang
protoc -I . proto/neuroevolution.proto --go_out=plugins=grpc:.
```

# Some policies

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 3663099671, 2948785697, 2098781916, 3154979704, 3257521261, 4184203103, 1885360337, 2130262694]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 1119816405, 1862569959, 4045600106, 3323378031, 3482377035, 2408647873, 992432518, 3136608193, 1829394318, 1543114441]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 3944723327, 2251549834, 551244736, 2659881976, 2723408499, 3125959212, 3024698394, 312341439, 1481533847, 1751213060, 2773192657, 1242493479, 4240398627, 1935903898, 4273316625, 120186444]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 3944723327, 2251549834, 551244736, 2659881976, 2723408499, 3125959212, 3024698394, 312341439, 1481533847, 1751213060, 2773192657, 1242493479, 4240398627, 1935903898, 4273316625]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 3663099671, 2948785697, 2098781916, 3154979704, 3257521261, 4184203103, 1546417913, 1190962580, 1519291590, 4142039378, 2791828317, 794877055, 1407797410, 1212357677, 2357465736]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 3663099671, 2948785697, 2098781916, 3154979704, 3257521261, 4184203103, 1546417913, 1190962580, 1519291590, 4142039378, 2791828317, 2885439578, 341938862, 1790688615, 2165249093]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 3663099671, 2948785697, 2098781916, 3154979704, 3257521261, 4184203103, 1546417913, 1190962580, 1519291590, 4142039378, 2087398015, 2923945039, 2925837673, 4223546693, 1184104221, 3272207894]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 3944723327, 2251549834, 551244736, 2659881976, 2723408499, 3125959212, 3024698394, 312341439, 1481533847, 1751213060, 2615306691, 812490633, 2078119438, 2090160661, 724766844, 3458845734]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2693349394, 3452885796, 1637163948, 1847219188, 3239734781, 546358686, 3663099671, 2948785697, 2098781916, 3154979704, 3257521261, 4184203103, 1546417913, 1190962580, 1519291590, 4142039378, 2791828317, 794877055, 1407797410, 1212357677, 2357465736, 2331237694, 155839330, 3730261797, 2730616978, 3595209610, 3230716243]

[4070460811, 2393459932, 3391677529, 3126527182, 1530841827, 551825296, 2788280626, 3259895649, 491621571, 1975069066, 2436286981, 1561675863, 1783350318, 1327606738, 2368546632, 1861319266, 2926076564, 3244028662, 770972465, 682803462, 4187104692, 1531762673, 227312423, 687820495, 2436347868, 1827742750, 2029378539, 983443351, 3317892383, 3181143904, 206189380, 1488772059, 1594572977, 1868808599, 2988289214, 4078764900]