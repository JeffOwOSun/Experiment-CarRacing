# Experiment-CarRacing


## Dependencies

We will use the [A3C algorithm](https://arxiv.org/abs/1602.01783) to solve the `CarRacing-v0` environment. The code presented here is based on [Chris Nicholls](https://github.com/cgnicholls/reinforcement-learning/tree/master/a3c), [Elibol and Khan's](https://github.com/oguzelibol/CarRacingA3C), and [Jang, Min and Lee's](https://github.com/sjang92/car_racing) implementations. The only difference will be that our implementation will be written specifically for Python 3.5 and upwards, as well as any dependency being equal to that of [Gym's universe-starter-agent](https://github.com/openai/universe-starter-agent).


## Neural Network Architecture

Choosing the correct architecture is a time-consuming task, but luckily this work has already been done for us. [Jang, Min and Lee's](https://www.scribd.com/document/358019044/Reinforcement-Car-Racing-with-A3C) tested from a two-layered CNN up to seven-layered CNN and found that the best results where obtained with the two-layered CNN. This was due mostly because: 

1) The states returned by Gym are too small, even compared to the Atari environments (`96*96*3` vs. `210*160*3`).
2) Deeper networks are harder to train, as the models would be more likely to fall into a local minima at the beginning of training. 
3) Deeper networks take longer to train, due to the larger number of parameters, and thus there is no way of telling when the model has converged.

While it is tempting to simply use the same neural network architecture as [Gym's universe-starter-agent](https://github.com/openai/universe-starter-agent), it won't perform well in this task for the reasons stated before, as this architecture consists of a four-layered CNN. Thus, we will choose the same architecture as Jang, Min and Lee, that is:

* Layer 1: F=16, W=8, S=4
* Layer 2: F=32, W=3, S=2

as per the notation used in [CS231n](http://cs231n.github.io/convolutional-networks/#conv).

## Action and State spaces
The continuity of the action and observation spaces are key characteristics of this Gym environment, which are perhaps what lead to the obscurity of the `Box2D` spaces as they are not as famous as the Atari 2600 games. Indeed, running:

```python
import gym
env = gym.make('CarRacing-v0')
env.action_space
env.observation_space
```

we obtain `Box(3,)` and `Box(96,96,3)`, respectively, so the `env.action_space.n` method in the universe-starter-agent does not work for us here. Digging further, the former implies that our actions are of the form `[steer, gas, brake]`, with `steer`, `gas` and `brake` being real numbers. However, not any number may be entered. Indeed, running:

```python
env.action_space.low
env.action_space.high
```

will yield, respectively, `array([-1, 0, 0])` and `array([1, 1, 1])`, so `-1<=steer<=1`, `0<=gas<=1` and `0<=brake<=1`. 

### Discretizing the action space

Due to both computing limitations and some combination of actions making no sense or being 'dangerous' (e.g. gas and braking at the same time, or accelerating and steering as explained in the [documentation of the environment](https://github.com/olegklimov/gym/blob/27f03a2014dd047745f307ee06c938e5a2818656/gym/envs/box2d/car_racing.py#L37-L38)), we will discretize the actions available to our agent. Indeed, we can try using the `MultiDiscrete` space like so:

```python
from gym import spaces
self.action_space = spaces.MultiDiscrete([[-1, 1], [0, 1], [0, 1]])
```

However, this would still present us with some 'forbidden' combinations of actions we have previously mentioned. As such, we will then limit our available actions to 5:

```python
self.action_space = [[1, 0, 0], [-1, 0, 0], [0, 1, 0], [0, 0, 0.8]]
```

that is, turn right, turn left, accelerate, and brake, respectively, where our brake is limited to `0.8` following the recommendation by [Elibol and Khan's](https://github.com/oguzelibol/CarRacingA3C) implementation. In the future we plan to ignore this limitation and use the `MultiDiscrete` action space, as it is always possible that our agent might find that some combinations of actions which seem nonsensical to us might be of use for specific scenarios.

#### Continuous Certainty

As explained in [Jang, Min and Lee's](https://www.scribd.com/document/358019044/Reinforcement-Car-Racing-with-A3C) implementation, we can address the issue of our agent having only 4 different actions to use at any given moment by simply taking the softmax probability of our action and multiply it by the action to be taken. 

For example, if we have at a given section in the track a softmax probability `0.24` to accelerate, and the softmax for the rest of the actions are all 0.19, then it doesn't make much sense for the agent to fully accelerate. Thus, what continuous certainty proposes is that the agent should accelerate `0.19*1.00=0.19`, that is, take the action `[0.0, 0.19, 0.0]`, or accelerate 19% of the capability of the car. The authors note that this is more how human drivers learn to drive when encountering a new area or setting, or effectively, when it is the first time for the driver to actually drive.

This addresses the issue when the agent is 'not sure enough' that the action to take is the correct one, and thus it is preferrable for the agent not to fully take said action. 

## Future Work

We will try to reproduce the work of Jang, Min and Lee, but looking at their implementation, we will do the following tests (changes and/or additions):

0) Add Continuous Certainty to mimic continuity in the possible actions taken by our agent.
1) Add specifically the action 'no action' to the action space, i.e., add `[0, 0, 0]` to our possible actions available.
2) As discussed previously, change our action space to `spaces.MultiDiscrete([[-1, 1], [0, 1], [0, 1]])`.
3) Use 2 threads insted of the planned 4.
4) To ensure long-term stability of our code, use `tf.layers` instead of `tf.contrib.layers` for our initializers, as well as the convolutional and fully connected layers.
5) Add a RNN after our final fully connected layer.
6) Use ELU instead of ReLU as our activation functions.

For comparison, we will use as a baseline Elibol and Khan's 100-episode score of 652.29 ± 10.17 and Jang, Min and Lee's score of 571.68 ± 19.38 and see whether these changes will improve or not the known result.

## Running

To run our agent, we enter the following in the command prompt:

```
python a3c_lstm.py -env <env_name> -s <save_path> -t <num_threads> -r <the T in save_path-t>
```

*Note:* this is our first version of the code, and as such it is most likely full of errors, as well as it being an adaptation of [Chris Nicholls's](https://github.com/cgnicholls/reinforcement-learning/tree/master/a3c) code. It will be slowly updated with the changes we make.
