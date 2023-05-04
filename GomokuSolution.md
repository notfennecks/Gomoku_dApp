```python
class AppVariables:
    PlayerXState = Bytes("PlayerXState") 
    PlayerOState = Bytes("PlayerOState")

    PlayerOAddress = Bytes("PlayerOAddress")
    PlayerXAddress = Bytes("PlayerXAddress")
    PlayerTurnAddress = Bytes("PlayerTurnAddress")
    FundsEscrowAddress = Bytes("FundsEscrowAddress")

    BetAmount = Bytes("BetAmount")
    ActionTimeout = Bytes("ActionTimeout")
    GameStatus = Bytes("GameState")

    @classmethod
    def number_of_int(cls):
        return 5

    @classmethod
    def number_of_str(cls):
        return 4

class DefaultValues:
    PlayerXState = Int(0)
    PlayerOState = Int(0)
    GameStatus = Int(0)
    BetAmount = Int(1000000)
    GameDurationInSeconds = Int(7200) #2 hours

class AppActions:
    SetupPlayers = Bytes("SetupPlayers")
    ActionMove = Bytes("ActionMove")
    MoneyRefund = Bytes("MoneyRefund")

def application_start():
    is_app_initialization = Txn.application_id() == Int(0)

    actions = Cond(
        [Txn.application_args[0] == AppActions.SetupPlayers, initialize_players_logic()],
        [And(Txn.application_args[0] == AppActions.ActionMove,
             Global.group_size() == Int(1)), play_action_logic()],
        [Txn.application_args[0] == AppActions.MoneyRefund, money_refund_logic()]
    )

    return If(is_app_initialization, app_initialization_logic(), actions)

def app_initialization_logic():
    return Seq([
        App.globalPut(AppVariables.PlayerXState, DefaultValues.PlayerXState),
        App.globalPut(AppVariables.PlayerOState, DefaultValues.PlayerOState),
        App.globalPut(AppVariables.GameStatus, DefaultValues.GameStatus),
        App.globalPut(AppVariables.BetAmount, DefaultValues.BetAmount),
        Return(Int(1))
    ])

def initialize_players_logic():
    player_x_address = App.globalGetEx(Int(0), AppVariables.PlayerXAddress)
    player_o_address = App.globalGetEx(Int(0), AppVariables.PlayerOAddress)

    setup_failed = Seq([
        Return(Int(0))
    ])

    setup_players = Seq([
        Assert(Gtxn[1].type_enum() == TxnType.Payment),
        Assert(Gtxn[2].type_enum() == TxnType.Payment),
        Assert(Gtxn[1].receiver() == Gtxn[2].receiver()),
        Assert(Gtxn[1].amount() == App.globalGet(AppVariables.BetAmount)),
        Assert(Gtxn[2].amount() == App.globalGet(AppVariables.BetAmount)),
        App.globalPut(AppVariables.PlayerXAddress, Gtxn[1].sender()),
        App.globalPut(AppVariables.PlayerOAddress, Gtxn[2].sender()),
        App.globalPut(AppVariables.PlayerTurnAddress, Gtxn[1].sender()),
        App.globalPut(AppVariables.FundsEscrowAddress, Gtxn[1].receiver()),
        App.globalPut(AppVariables.ActionTimeout, Global.latest_timestamp() + DefaultValues.GameDurationInSeconds),
        Return(Int(1))
    ])

    return Seq([
        player_x_address,
        player_o_address,
        If(Or(player_x_address.hasValue(), player_o_address.hasValue()), setup_failed, setup_players)
    ])

#Logic for checking winner by 4 in a row. Cannot use board states to check for winner as the amount of states needed to check are extremely high. As with any tictactoe game the more dimensions you add to the board the amount of winning states and game staes increase exponentially. Way easier to check directly if a player has one by analyizing pieces on the board.
def has_player_won(board)
    #returns either 'X' or 'O' which is the winner
    # check rows
    for row in range(board_size):
        for col in range(board_size - winning_length + 1):
            if board[row][col] != empty and \
               board[row][col] == board[row][col+1] and \
               board[row][col] == board[row][col+2] and \
               board[row][col] == board[row][col+3]:
                return board[row][col]
    # check columns
    for row in range(board_size - winning_length + 1):
        for col in range(board_size):
            if board[row][col] != empty and \
               board[row][col] == board[row+1][col] and \
               board[row][col] == board[row+2][col] and \
               board[row][col] == board[row+3][col]:
                return board[row][col]
    # check diagonal
    for row in range(board_size - winning_length + 1):
        for col in range(board_size - winning_length + 1):
            if board[row][col] != empty and \
               board[row][col] == board[row+1][col+1] and \
               board[row][col] == board[row+2][col+2] and \
               board[row][col] == board[row+3][col+3]:
                return board[row][col]
    # check reverse diagonal
    for row in range(board_size - winning_length + 1):
        for col in range(winning_length - 1, board_size):
            if board[row][col] != empty and \
               board[row][col] == board[row+1][col-1] and \
               board[row][col] == board[row+2][col-2] and \
               board[row][col] == board[row+3][col-3]:
                return board[row][col]
    return empty

def is_tie():
    #if state is 1267650600228229401496703205375. Which is 100 1s in binary, so every space is filled. The game is a tie.
    return if(state == 1267650600228229401496703205375)

def play_action_logic():
    position_index = Btoi(Txn.application_args[1])

    state_x = App.globalGet(AppVariables.PlayerXState)
    state_o = App.globalGet(AppVariables.PlayerOState)

    #Check if player chosen position is next to a current piece
    #If position index + [0][1] != empty (right)
    #|| position index + [0][-1] != empty (left)
    #|| position index + [-1][0] != empty (up)
    #|| position index + [1][0] != empty (down)
    #|| position index + [-1][1] != empty (top right)
    #|| position index + [-1][-1] != empty (top left)
    #|| position index + [1][1] != empty (bottom right)
    #|| position index + [1][-1] != empty (bottom left)
    game_action = ShiftLeft(Int(1), position_index) # activate the bit at position "position_index"

    player_x_move = Seq([
        App.globalPut(AppVariables.PlayerXState, BitwiseOr(state_x, game_action)), # fill the game_action bit

        If(has_player_won(App.globalGet(AppVariables.PlayerXState)),
           App.globalPut(AppVariables.GameStatus, Int(1))), # update the game status in case of a win

        App.globalPut(AppVariables.PlayerTurnAddress, App.globalGet(AppVariables.PlayerOAddress)), 
    ])

    player_o_move = Seq([
        App.globalPut(AppVariables.PlayerOState, BitwiseOr(state_o, game_action)),

        If(has_player_won(App.globalGet(AppVariables.PlayerOState)),
           App.globalPut(AppVariables.GameStatus, Int(2))),

        App.globalPut(AppVariables.PlayerTurnAddress, App.globalGet(AppVariables.PlayerXAddress)),
    ])

    return Seq([
        Assert(position_index >= Int(0)), # valid position interval, top left position
        Assert(position_index <= Int(99)), # valid position interval, bottom right position
        Assert(Global.latest_timestamp() <= App.globalGet(AppVariables.ActionTimeout)), # valid time interval
        Assert(App.globalGet(AppVariables.GameStatus) == DefaultValues.GameStatus), # is game active
        Assert(Txn.sender() == App.globalGet(AppVariables.PlayerTurnAddress)), # valid player
        Assert(And(BitwiseAnd(state_x, game_action) == Int(0),
                   BitwiseAnd(state_o, game_action) == Int(0))), # the i-th position in the board is empty
        Cond(
            [Txn.sender() == App.globalGet(AppVariables.PlayerXAddress), player_x_move],
            [Txn.sender() == App.globalGet(AppVariables.PlayerOAddress), player_o_move],
        ),
        If(is_tie(), App.globalPut(AppVariables.GameStatus, Int(3))), # adjust the status in case of a tie.
        Return(Int(1))
    ])

def money_refund_logic():
    has_x_won_by_playing = App.globalGet(AppVariables.GameStatus) == Int(1) # normal win by placing marks
    has_o_won_by_playing = App.globalGet(AppVariables.GameStatus) == Int(2) # normal win by placing marks

    has_x_won_by_timeout = And(App.globalGet(AppVariables.GameStatus) == Int(0),
                               Global.latest_timestamp() > App.globalGet(AppVariables.ActionTimeout),
                               App.globalGet(AppVariables.PlayerTurnAddress) == App.globalGet(
                                   AppVariables.PlayerOAddress)) # win by timeout logic.

    has_o_won_by_timeout = And(App.globalGet(AppVariables.GameStatus) == Int(0),
                               Global.latest_timestamp() > App.globalGet(AppVariables.ActionTimeout),
                               App.globalGet(AppVariables.PlayerTurnAddress) == App.globalGet(
                                   AppVariables.PlayerXAddress)) # win by timeout logic.

    has_x_won = Or(has_x_won_by_playing, has_x_won_by_timeout) # anykind of win
    has_o_won = Or(has_o_won_by_playing, has_o_won_by_timeout) # anykind of win
    game_is_tie = App.globalGet(AppVariables.GameStatus) == Int(3) 

    x_withdraw = Seq([
        Assert(Gtxn[1].receiver() == App.globalGet(AppVariables.PlayerXAddress)), 
        Assert(Gtxn[1].amount() == Int(2) * App.globalGet(AppVariables.BetAmount)),
        App.globalPut(AppVariables.GameStatus, Int(1)) 
    ])

    o_withdraw = Seq([
        Assert(Gtxn[1].receiver() == App.globalGet(AppVariables.PlayerOAddress)),
        Assert(Gtxn[1].amount() == Int(2) * App.globalGet(AppVariables.BetAmount)),
        App.globalPut(AppVariables.GameStatus, Int(2))
    ])

    tie_withdraw = Seq([
        Assert(Gtxn[1].receiver() == App.globalGet(AppVariables.PlayerXAddress)),
        Assert(Gtxn[1].amount() == App.globalGet(AppVariables.BetAmount)),
        Assert(Gtxn[2].type_enum() == TxnType.Payment),
        Assert(Gtxn[2].sender() == App.globalGet(AppVariables.FundsEscrowAddress)),
        Assert(Gtxn[2].receiver() == App.globalGet(AppVariables.PlayerOAddress)),
        Assert(Gtxn[2].amount() == App.globalGet(AppVariables.BetAmount))
    ])

    return Seq([
        Assert(Gtxn[1].type_enum() == TxnType.Payment),
        Assert(Gtxn[1].sender() == App.globalGet(AppVariables.FundsEscrowAddress)),
        Cond(
            [has_x_won, x_withdraw],
            [has_o_won, o_withdraw],
            [game_is_tie, tie_withdraw]
        ),
        Return(Int(1))
    ])

def approval_program():
    return application_start()

def clear_program():
    return Return(Int(1))

def game_funds_escorw(app_id: int):
    win_refund = Seq([
        Assert(Gtxn[0].application_id() == Int(app_id)),
        Assert(Gtxn[1].fee() <= Int(1000)),
        Assert(Gtxn[1].asset_close_to() == Global.zero_address()),
        Assert(Gtxn[1].rekey_to() == Global.zero_address())
    ])

    tie_refund = Seq([
        Assert(Gtxn[0].application_id() == Int(app_id)),
        Assert(Gtxn[1].fee() <= Int(1000)),
        Assert(Gtxn[1].asset_close_to() == Global.zero_address()),
        Assert(Gtxn[1].rekey_to() == Global.zero_address()),
        Assert(Gtxn[2].fee() <= Int(1000)),
        Assert(Gtxn[2].asset_close_to() == Global.zero_address()),
        Assert(Gtxn[2].rekey_to() == Global.zero_address())
    ])

    return Seq([
        Cond(
            [Global.group_size() == Int(2), win_refund],
            [Global.group_size() == Int(3), tie_refund],
        ),
        Return(Int(1))
    ])

class GameEngineService:
    def __init__(self,
                 app_creator_pk,
                 app_creator_address,
                 player_x_pk,
                 player_x_address,
                 player_o_pk,
                 player_o_address):
        self.app_creator_pk = app_creator_pk
        self.app_creator_address = app_creator_address
        self.player_x_pk = player_x_pk
        self.player_x_address = player_x_address
        self.player_o_pk = player_o_pk
        self.player_o_address = player_o_address
        self.teal_version = 4

        self.approval_program_code = approval_program()
        self.clear_program_code = clear_program()

        self.app_id = None
        self.escrow_fund_address = None
        self.escrow_fund_program_bytes = None

def deploy_application(self, client):
    approval_program_compiled = compileTeal(approval_program(),
                                            mode=Mode.Application,
                                            version=self.teal_version)

    clear_program_compiled = compileTeal(clear_program(),
                                         mode=Mode.Application,
                                         version=self.teal_version)

    approval_program_bytes = NetworkInteraction.compile_program(client=client,
                                                                source_code=approval_program_compiled)

    clear_program_bytes = NetworkInteraction.compile_program(client=client,
                                                             source_code=clear_program_compiled)

    global_schema = algo_txn.StateSchema(num_uints=AppVariables.number_of_int(),
                                         num_byte_slices=AppVariables.number_of_str())

    local_schema = algo_txn.StateSchema(num_uints=0,
                                        num_byte_slices=0)

    app_transaction = \
    	ApplicationTransactionRepository.create_application(client=client,
                                                          creator_private_key=self.app_creator_pk,
                                                          approval_program=approval_program_bytes,
                                                          clear_program=clear_program_bytes,
                                                          global_schema=global_schema,
                                                          local_schema=local_schema,
                                                          app_args=None)

    tx_id = NetworkInteraction.submit_transaction(client,
                                                  transaction=app_transaction,
                                                  log=False)

    transaction_response = client.pending_transaction_info(tx_id)

    self.app_id = transaction_response['application-index']
    print(f'Gomoku application deployed with the application_id: {self.app_id}')

def start_game(self, client):
    """
    Atomic transfer of 3 transactions:
    - 1. Application call
    - 2. Payment from the Player X address to the Escrow fund address
    - 3. Payment from the Player O address to the Escrow fund address
    """
    if self.app_id is None:
        raise ValueError('The application has not been deployed')

    if self.escrow_fund_address is not None or self.escrow_fund_program_bytes is not None:
        raise ValueError('The game has already started!')

    escrow_fund_program_compiled = compileTeal(game_funds_escorw(app_id=self.app_id),
                                               mode=Mode.Signature,
                                               version=self.teal_version)

    self.escrow_fund_program_bytes = \
    								NetworkInteraction.compile_program(client=client,
                                       								 source_code=escrow_fund_program_compiled)

    self.escrow_fund_address = algo_logic.address(self.escrow_fund_program_bytes)

    player_x_funding_txn = PaymentTransactionRepository.payment(client=client,
                                                                sender_address=self.player_x_address,
                                                                receiver_address=self.escrow_fund_address,
                                                                amount=1000000,
                                                                sender_private_key=None,
                                                                sign_transaction=False)

    player_o_funding_txn = PaymentTransactionRepository.payment(client=client,
                                                                sender_address=self.player_o_address,
                                                                receiver_address=self.escrow_fund_address,
                                                                amount=1000000,
                                                                sender_private_key=None,
                                                                sign_transaction=False)

    app_args = [
        "SetupPlayers"
    ]

    app_initialization_txn = \
        ApplicationTransactionRepository.call_application(client=client,
                                                          caller_private_key=self.app_creator_pk,
                                                          app_id=self.app_id,
                                                          on_complete=algo_txn.OnComplete.NoOpOC,
                                                          app_args=app_args,
                                                          sign_transaction=False)

    gid = algo_txn.calculate_group_id([app_initialization_txn,
                                       player_x_funding_txn,
                                       player_o_funding_txn])

    app_initialization_txn.group = gid
    player_x_funding_txn.group = gid
    player_o_funding_txn.group = gid

    app_initialization_txn_signed = app_initialization_txn.sign(self.app_creator_pk)
    player_x_funding_txn_signed = player_x_funding_txn.sign(self.player_x_pk)
    player_o_funding_txn_signed = player_o_funding_txn.sign(self.player_o_pk)

    signed_group = [app_initialization_txn_signed,
                    player_x_funding_txn_signed,
                    player_o_funding_txn_signed]

    txid = client.send_transactions(signed_group)

    print(f'Game started with the transaction_id: {txid}')

def play_action(self, client, player_id: str, action_position: int):
    app_args = [
        "ActionMove",
        action_position]

    player_pk = self.player_x_pk if player_id == "X" else self.player_o_pk

    app_initialization_txn = \
        ApplicationTransactionRepository.call_application(client=client,
                                                          caller_private_key=player_pk,
                                                          app_id=self.app_id,
                                                          on_complete=algo_txn.OnComplete.NoOpOC,
                                                          app_args=app_args)

    tx_id = NetworkInteraction.submit_transaction(client,
                                                  transaction=app_initialization_txn,
                                                  log=False)

    print(f'{player_id} has been put at position {action_position} in transaction with id: {tx_id}')

def win_money_refund(self, client, player_id: str):
    """
    Atomic transfer of 2 transactions:
    1. Application call
    2. Payment from the Escrow account to winner address either PlayerX or PlayerO.
    """
    player_pk = self.player_x_pk if player_id == "X" else self.player_o_pk
    player_address = self.player_x_address if player_id == "X" else self.player_o_address

    app_args = [
        "MoneyRefund"
    ]

    app_withdraw_call_txn = \
        ApplicationTransactionRepository.call_application(client=client,
                                                          caller_private_key=player_pk,
                                                          app_id=self.app_id,
                                                          on_complete=algo_txn.OnComplete.NoOpOC,
                                                          app_args=app_args,
                                                          sign_transaction=False)

    refund_txn = PaymentTransactionRepository.payment(client=client,
                                                      sender_address=self.escrow_fund_address,
                                                      receiver_address=player_address,
                                                      amount=2000000,
                                                      sender_private_key=None,
                                                      sign_transaction=False)

    gid = algo_txn.calculate_group_id([app_withdraw_call_txn,
                                       refund_txn])

    app_withdraw_call_txn.group = gid
    refund_txn.group = gid

    app_withdraw_call_txn_signed = app_withdraw_call_txn.sign(player_pk)

    refund_txn_logic_signature = algo_txn.LogicSig(self.escrow_fund_program_bytes)
    refund_txn_signed = algo_txn.LogicSigTransaction(refund_txn, refund_txn_logic_signature)

    signed_group = [app_withdraw_call_txn_signed,
                    refund_txn_signed]

    txid = client.send_transactions(signed_group)

    print(f'The winning money have been refunded to the player {player_id} in the transaction with id: {txid}')

def tie_money_refund(self, client):
    """
    Atomic transfer of 3 transactions:
    1. Application call
    2. Payment from the escrow address to the PlayerX address.
    3. Payment from the escrow address to the PlayerO address.
    """
    if self.app_id is None:
        raise ValueError('The application has not been deployed')

    app_args = [
        "MoneyRefund"
    ]

    app_withdraw_call_txn = \
        ApplicationTransactionRepository.call_application(client=client,
                                                          caller_private_key=self.app_creator_pk,
                                                          app_id=self.app_id,
                                                          on_complete=algo_txn.OnComplete.NoOpOC,
                                                          app_args=app_args,
                                                          sign_transaction=False)

    refund_player_x_txn = PaymentTransactionRepository.payment(client=client,
                                                               sender_address=self.escrow_fund_address,
                                                               receiver_address=self.player_x_address,
                                                               amount=1000000,
                                                               sender_private_key=None,
                                                               sign_transaction=False)

    refund_player_o_txn = PaymentTransactionRepository.payment(client=client,
                                                               sender_address=self.escrow_fund_address,
                                                               receiver_address=self.player_o_address,
                                                               amount=1000000,
                                                               sender_private_key=None,
                                                               sign_transaction=False)

    gid = algo_txn.calculate_group_id([app_withdraw_call_txn,
                                       refund_player_x_txn,
                                       refund_player_o_txn])

    app_withdraw_call_txn.group = gid
    refund_player_x_txn.group = gid
    refund_player_o_txn.group = gid

    app_withdraw_call_txn_signed = app_withdraw_call_txn.sign(self.app_creator_pk)

    refund_player_x_txn_logic_signature = algo_txn.LogicSig(self.escrow_fund_program_bytes)
    refund_player_x_txn_signed = \
        algo_txn.LogicSigTransaction(refund_player_x_txn, refund_player_x_txn_logic_signature)

    refund_player_o_txn_logic_signature = algo_txn.LogicSig(self.escrow_fund_program_bytes)
    refund_player_o_txn_signed = \
        algo_txn.LogicSigTransaction(refund_player_o_txn, refund_player_o_txn_logic_signature)

    signed_group = [app_withdraw_call_txn_signed,
                    refund_player_x_txn_signed,
                    refund_player_o_txn_signed]

    txid = client.send_transactions(signed_group)

    print(f'The initial bet money have been refunded to the players in the transaction with id: {txid}')
```
