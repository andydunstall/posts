# Implementing TD-Gammon with Keras
TD-Gammon is an artificial neural network, trained with TD(λ), that learns to play Backgammon by self-play.

This post implements TD-Gammon 0.0 in Python using Keras and Tensorflow. A full description of the TD-Gammon is given in [Temporal Difference Learning and TD-Gammon](https://pdfs.semanticscholar.org/917e/e68192527f0722fac966163f26b7a4e8e5f3.pdf?_ga=2.138006640.1591278561.1609908105-703813112.1609908105) and [Practical Issues in Temporal Difference Learning](https://papers.nips.cc/paper/1991/file/68ce199ec2c5517597ce0a4d89620f55-Paper.pdf). A case study of TD-Gammon is also given in Reinforcement Learning: An Introduction (Sutton and Barto).

The code for this implementation can be found on [GitHub](https://github.com/andydunstall/td-gammon).

## Neural Network and Action Selection
The neural network is used to estimate the value of each state (board position). This takes the board position as input and outputs a real value between 0 and 1 using a sigmoid activation function. During training the reward for each state is 0, except if player one won, in which case the reward is 1. Therefore the value function can be interpreted as the probability of player one winning starting from that state (so the probability of player two winning is simply 1 minus the value function).

Each turn of the TD-Gammon agent:
1. Rolls the dice,
2. Finds all permitted moves given the current board position and rolls of the dice,
3. Calculates the afterstate for all permitted moves (the new board position after each move),
4. Calculates the value of each afterstate given by the network,
5. Then finally plays the move that leads to the afterstate with the highest probability of winning.

Therefore this is simply a greedy policy with respect to the value function given by the network.

```python
def action(self, board, roll, player):
    """Predicts the optimal move given the current state.

    This calculates each afterstate for all possible moves given the
    current state and selects the action that leads to the state with
    the greatest afterstate value.

    Arguments:
    board -- board containing the game state
    roll -- list of dice rolls left in the players turn
    player -- number of the player
    """
    start = time.time()

    max_move = None
    max_prob = -np.inf
    permitted = board.permitted_moves(roll, player)
    for move in permitted:
        afterstate = copy.deepcopy(board)
        if not afterstate.move(*move, player):
            logging.error("model requested an invalid move")
            continue

        state = afterstate.encode_state(player)[np.newaxis]
        prob = tf.reduce_sum(self._model(state))
        # The network gives the probability of player 0 winning so must
        # change if player 1.
        prob = 1 - prob if player == 1 else prob
        if prob > max_prob:
            max_prob = prob
            max_move = move

    if self._state is None:
        self._state = tf.Variable(board.encode_state(player))
    if self._value is None:
        self._value = tf.Variable(self._model(self._state[np.newaxis]))

    duration = time.time() - start
    logging.debug("playing move [player = %d] [move = %s] [winning prob = %f] [duration = %ds]", player, str(max_move), max_prob, duration)
    return max_move
```

The original paper describes a network with 4 outputs corresponding to player one or player two winning either a normal win or a gammon, though I’m using the network described in Sutton and Barto which has a single output.

The board position is encoded as 198 input units encoding each player’s checkers positions on the board, on the bar and removed from the board. Also two additional units encoding which player is next to play.

The network has a single hidden layer of 40 units and uses the sigmoid activation function. Note later versions use more hidden units though this only implements TD-Gammon 0.0.

```python
def __init__(self, restore_path = None):
    """Construct a model with random weights.

    Arguments:
    restore_path -- path to stored checkpoint to restore if given
        (default None)
    """
    inputs = tf.keras.Input(shape=(198,))
    x = tf.keras.layers.Dense(40, activation="sigmoid")(inputs)
    outputs = tf.keras.layers.Dense(1, activation="sigmoid")(x)
    self._model = tf.keras.Model(inputs=inputs, outputs=outputs)
    // ...
```

## Training
The model learns by playing many games against itself. After each move the model will calculate the gradient of the value of the new state with respect to the network weights.

The TD error can then be calculated as the reward plus the value of the new state minus the value of the previous state.

Finally the eligibility trace can be updated and applied to the weights.

```python
def update(self, board, player):
    """Updates the model given the current state and reward.

    This is expected to be called after the player has made their move.

    The aim is to move the predicted values towards the actual reward
    using TD-lambda.

    Arguments:
    board -- board containing the game state
    roll -- list of dice rolls left in the players turn
    """
    start = time.time()

    x_next = board.encode_state(player)
    with tf.GradientTape() as tape:
        value_next = self._model(x_next[np.newaxis])

    trainable_vars = self._model.trainable_variables
    grads = tape.gradient(value_next, trainable_vars)

    # Lazily initialize when gradient shape known.
    if len(self._trace) == 0:
        for grad in grads:
            self._trace.append(tf.Variable(
                tf.zeros(grad.get_shape()), trainable=False
            ))

    if player == 0 and board.won(player):
        reward = 1
    else:
        reward = 0

    td_error = tf.reduce_sum(reward + value_next - self._value)
    for i in range(len(grads)):
        self._trace[i].assign((self._LAMBDA * self._trace[i]) + grads[i])

        grad_trace = self._ALPHA * td_error * self._trace[i]
        self._model.trainable_variables[i].assign_add(grad_trace)

    self._state = tf.Variable(x_next)
    self._value = tf.Variable(value_next)

    duration = time.time() - start
    logging.debug("updating model [player = %d] [duration = %ds]", player, duration)
```

Note alpha and lambda are given by [Practical Issues in Temporal Difference Learning and TD-Gammon](https://pdfs.semanticscholar.org/917e/e68192527f0722fac966163f26b7a4e8e5f3.pdf?_ga=2.138006640.1591278561.1609908105-703813112.1609908105) as 0.7 and 0.1 respectively, and a discount rate of 1 is used.

## Evaluation
To keep evaluation very simple, the TD-Gammon agent plays 1000 games against an agent selecting actions randomly from the permitted moves each turn. Running notebook.ipynb in AWS SageMaker for 500 training episodes, which took around a day (on a ml.t2.medium instance) as each episode is a few thousand moves.

After 500 training episodes the TD-Gammon agent won 97.5% of its games.
