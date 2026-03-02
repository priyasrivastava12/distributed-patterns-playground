LUDO

We will include:

* Clean architecture
* Proper responsibilities
* SOLID principles
* Extensible rule engine
* Design patterns
* Sequence diagram
* Code structure

Patterns used:

* **Singleton → Dice**
* **Factory → Player**
* **State → Token lifecycle**
* **Strategy → Move rules**
* **Command → Moves**
* **Observer → Game events**
* **Chain of Responsibility → Move validation**

---

# 1. High Level Architecture

```text
                     +------------------+
                     |       Game       |
                     +------------------+
                             |
                             v
                     +------------------+
                     |   TurnManager    |
                     +------------------+
                             |
                             v
                     +------------------+
                     |   MoveExecutor   |
                     +------------------+
                             |
                ---------------------------------
                |               |               |
                v               v               v
          MoveValidator     RuleEngine        Board
       (Chain Pattern)   (Strategy Pattern)
                |
                v
            TokenState
          (State Pattern)

Dice (Singleton)

GameEventManager (Observer)
```

---

# 2. Core Responsibilities

### Game

Orchestrates everything.

### TurnManager

Handles player turns.

### MoveExecutor

Executes validated moves.

### MoveValidator

Validates moves before execution.

### RuleEngine

Applies game rules.

### Board

Maintains cells and tokens.

### TokenState

Controls token behavior.

---

# 3. Game Class

```java
class Game {

    private Board board;
    private TurnManager turnManager;
    private MoveExecutor moveExecutor;
    private Dice dice;

    public Game(List<Player> players){

        this.board = new Board();
        this.turnManager = new TurnManager(players);
        this.moveExecutor = new MoveExecutor(board);
        this.dice = Dice.getInstance();
    }

    public void startGame(){

        while(true){

            Player player = turnManager.getCurrentPlayer();

            int diceValue = dice.roll();

            Token token = player.chooseToken(diceValue);

            Move move = new Move(player,token,diceValue);

            moveExecutor.executeMove(move);

            if(player.hasWon()){
                System.out.println(player.getName()+" wins");
                break;
            }

            turnManager.nextTurn(diceValue);
        }
    }
}
```

---

# 4. Move Object

Encapsulates move details.

```java
class Move {

    Player player;
    Token token;
    int diceValue;

    public Move(Player player,Token token,int dice){

        this.player = player;
        this.token = token;
        this.diceValue = dice;
    }
}
```

---

# 5. MoveExecutor

Executes move using **Command pattern**.

```java
class MoveExecutor {

    private Board board;
    private MoveValidator validator;

    public MoveExecutor(Board board){

        this.board = board;
        this.validator = new MoveValidator();
    }

    public void executeMove(Move move){

        if(!validator.validate(move))
            return;

        MoveCommand command =
                new TokenMoveCommand(move,board);

        command.execute();
    }
}
```

---

# 6. MoveValidator (Chain of Responsibility)

Each validator checks a rule.

```java
interface MoveValidationRule {

    boolean validate(Move move);
}
```

---

### DiceRule

```java
class DiceRule implements MoveValidationRule {

    public boolean validate(Move move){

        return move.diceValue >=1 && move.diceValue <=6;
    }
}
```

---

### TokenMoveRule

```java
class TokenMoveRule implements MoveValidationRule {

    public boolean validate(Move move){

        return move.token.canMove(move.diceValue);
    }
}
```

---

### MoveValidator

```java
class MoveValidator {

    List<MoveValidationRule> rules;

    public MoveValidator(){

        rules = List.of(
                new DiceRule(),
                new TokenMoveRule()
        );
    }

    public boolean validate(Move move){

        for(MoveValidationRule rule:rules){

            if(!rule.validate(move))
                return false;
        }

        return true;
    }
}
```

---

# 7. Command Pattern

```java
interface MoveCommand {

    void execute();
}
```

---

### TokenMoveCommand

```java
class TokenMoveCommand implements MoveCommand {

    private Move move;
    private Board board;

    public TokenMoveCommand(Move move,Board board){

        this.move = move;
        this.board = board;
    }

    public void execute(){

        Token token = move.token;

        token.move(move.diceValue,board);
    }
}
```

---

# 8. Token State Pattern

States:

```
BASE
ACTIVE
HOME
```

---

### TokenState

```java
interface TokenState {

    void move(Token token,int dice,Board board);

    boolean canMove(int dice);
}
```

---

### BaseState

```java
class BaseState implements TokenState {

    public void move(Token token,int dice,Board board){

        if(dice==6){

            token.setPosition(0);
            token.setState(new ActiveState());
        }
    }

    public boolean canMove(int dice){
        return dice==6;
    }
}
```

---

### ActiveState

```java
class ActiveState implements TokenState {

    public void move(Token token,int dice,Board board){

        int newPos = token.getPosition()+dice;

        board.moveToken(token,newPos);

        if(newPos >=57)
            token.setState(new HomeState());
    }

    public boolean canMove(int dice){
        return true;
    }
}
```

---

# 9. Board

```java
class Board {

    List<Cell> cells;

    public Board(){

        cells = new ArrayList<>();

        for(int i=0;i<52;i++)
            cells.add(new Cell(i,isSafeCell(i)));
    }

    public void moveToken(Token token,int pos){

        Cell cell = cells.get(pos%52);

        if(!cell.isSafe()){

            for(Token t:cell.getTokens()){

                if(t.getOwner()!=token.getOwner())
                    t.reset();
            }
        }

        cell.addToken(token);
    }

    private boolean isSafeCell(int pos){

        return pos==0||pos==8||pos==13||pos==21||
               pos==26||pos==34||pos==39||pos==47;
    }
}
```

---

# 10. TurnManager

```java
class TurnManager {

    Queue<Player> queue;

    public TurnManager(List<Player> players){

        queue = new LinkedList<>(players);
    }

    public Player getCurrentPlayer(){

        return queue.peek();
    }

    public void nextTurn(int dice){

        Player p = queue.poll();

        if(dice!=6)
            queue.offer(p);
        else
            queue.add(p);
    }
}
```

---

# 11. Observer Pattern

```java
interface GameEventListener {

    void onEvent(GameEvent event);
}
```

---

### EventManager

```java
class GameEventManager {

    List<GameEventListener> listeners;

    public void notify(GameEvent event){

        for(GameEventListener l:listeners)
            l.onEvent(event);
    }
}
```

---

# 12. Sequence Diagram (FAANG Level)

```text
Player
   |
   v
Game
   |
   | rollDice
   v
Dice (Singleton)
   |
   v
Game
   |
   | createMove
   v
MoveExecutor
   |
   v
MoveValidator (Chain of Responsibility)
   |
   v
TokenMoveCommand (Command Pattern)
   |
   v
Token
   |
   v
TokenState (State Pattern)
   |
   v
Board
   |
   v
Cell
   |
   v
GameEventManager (Observer)
```

---

# 13. Sequence Diagram (ASCII)

```text
Player    Game    Dice   Executor  Validator   Command   Token   Board
  |        |       |        |          |          |        |        |
  |------->|       |        |          |          |        |        |
  |        |----roll------->|          |          |        |        |
  |        |<---value-------|          |          |        |        |
  |        |----move------->|          |          |        |        |
  |        |                |--validate|          |        |        |
  |        |                |--------->|          |        |        |
  |        |                |<--ok-----|          |        |        |
  |        |                |--execute------------>|        |        |
  |        |                |                      |--move->|        |
  |        |                |                      |        |--cell->|
```

---

# 14. Clean Project Structure

```
ludo
 ├── model
 │     Game
 │     Player
 │     Token
 │     Move
 │     Board
 │     Cell
 │
 ├── engine
 │     MoveExecutor
 │     RuleEngine
 │
 ├── validation
 │     MoveValidator
 │     DiceRule
 │     TokenMoveRule
 │
 ├── command
 │     MoveCommand
 │     TokenMoveCommand
 │
 ├── state
 │     TokenState
 │     BaseState
 │     ActiveState
 │     HomeState
 │
 ├── observer
 │     GameEvent
 │     GameEventManager
 │
 └── util
       Dice
```

---

# 15. Why This Design Is Strong

This design:

✔ separates **validation, execution, and rules**
✔ supports **new rules easily**
✔ supports **undo/replay**
✔ supports **multiplayer servers**
✔ follows **SOLID principles**

---------------------------------------------------------------------------------



Below is a **complete FAANG-level LUDO LLD** suitable for **7+ years experience**.
It includes:

* Requirements
* Entities
* Class design
* Important design patterns
* Execution flow
* Sequence diagram
* Activity diagram
* Code structure

I will keep it **clean and realistic like a real backend system design**.

---

# 1. Functional Requirements

### Core Features

* 2–4 players
* Each player has **4 tokens**
* Dice roll (1-6)
* Token enters board on **6**
* Turn based
* Token capture
* Safe cells
* Win condition (all tokens reach home)

---

# 2. Important Design Patterns

| Pattern       | Why Used             |
| ------------- | -------------------- |
| **Strategy**  | Token movement rules |
| **State**     | Game states          |
| **Factory**   | Token creation       |
| **Command**   | Move representation  |
| **Observer**  | UI updates           |
| **Singleton** | GameManager          |

---

# 3. High Level Architecture

```text
Player
  |
  v
GameManager
  |
  v
Game
  |
  +---- DiceService
  |
  +---- MoveValidator
  |
  +---- MoveExecutor
  |
  v
Board
  |
  v
Cells
  |
  v
Tokens
```

---

# 4. Core Entities

```text
Game
Board
Cell
Player
Token
Dice
Move
Position
GameManager
```

---

# 5. Class Diagram (Conceptual)

```text
Game
 ├── List<Player>
 ├── Board
 ├── Dice
 ├── MoveValidator
 ├── MoveExecutor
 └── GameState

Player
 ├── id
 ├── color
 └── List<Token>

Token
 ├── id
 ├── color
 ├── Position
 └── TokenState

Board
 ├── List<Cell>
 └── moveToken()

Move
 ├── player
 ├── token
 ├── steps
 └── source/destination
```

---

# 6. Enums

```java
enum Color {
    RED, GREEN, BLUE, YELLOW
}

enum TokenState {
    BASE,
    ACTIVE,
    HOME
}

enum GameState {
    WAITING,
    ACTIVE,
    FINISHED
}
```

---

# 7. Position

```java
class Position {

    int index;

    Position(int index){
        this.index = index;
    }
}
```

---

# 8. Token

```java
class Token {

    String id;

    Color color;

    Position position;

    TokenState state;

    public boolean isAtBase(){
        return state == TokenState.BASE;
    }

}
```

---

# 9. Player

```java
class Player {

    String id;

    Color color;

    List<Token> tokens;

}
```

---

# 10. Cell

```java
class Cell {

    int index;

    boolean safe;

    List<Token> tokens = new ArrayList<>();

}
```

---

# 11. Board

```java
class Board {

    List<Cell> cells = new ArrayList<>();

    public Cell getCell(int index){
        return cells.get(index);
    }

}
```

Board typically contains **52 outer cells + home paths**.

---

# 12. Dice

```java
class Dice {

    Random random = new Random();

    public int roll(){
        return random.nextInt(6) + 1;
    }

}
```

---

# 13. Move (Command Pattern)

Move represents **player action**.

```java
class Move {

    Player player;

    Token token;

    int diceValue;

    int from;

    int to;

}
```

---

# 14. Move Strategy (Strategy Pattern)

Different rules depending on token state.

```java
interface MoveStrategy {

    int nextPosition(Token token, int dice);

}
```

---

# 15. DefaultMoveStrategy

```java
class DefaultMoveStrategy implements MoveStrategy {

    public int nextPosition(Token token, int dice){

        return token.position.index + dice;
    }

}
```

---

# 16. MoveValidator

Validates move rules.

```java
class MoveValidator {

    boolean isValidMove(Game game, Move move){

        Token token = move.token;

        if(token.state == TokenState.BASE && move.diceValue != 6)
            return false;

        return true;
    }

}
```

---

# 17. MoveExecutor

Executes token movement.

```java
class MoveExecutor {

    void execute(Game game, Move move){

        Board board = game.getBoard();

        Cell from = board.getCell(move.from);
        Cell to = board.getCell(move.to);

        from.tokens.remove(move.token);
        to.tokens.add(move.token);

        move.token.position = new Position(move.to);

    }

}
```

---

# 18. Game

Main orchestrator.

```java
class Game {

    Board board;

    List<Player> players;

    Dice dice;

    MoveValidator validator;

    MoveExecutor executor;

    Player currentPlayer;

}
```

---

# 19. Game Flow

```java
public void playTurn(Token token){

    int diceValue = dice.roll();

    int from = token.position.index;

    int to = from + diceValue;

    Move move = new Move(currentPlayer, token, diceValue, from, to);

    if(!validator.isValidMove(this, move))
        throw new RuntimeException("Invalid Move");

    executor.execute(this, move);

    switchTurn();

}
```

---

# 20. Turn Switching

```java
void switchTurn(){

    int index = players.indexOf(currentPlayer);

    currentPlayer = players.get((index+1) % players.size());

}
```

---

# 21. Sequence Diagram (Ludo Move)

```text
Player
 |
 | rollDice()
 v
Game
 |
 v
Dice.roll()
 |
 v
Game
 |
 | chooseToken()
 v
Game.playTurn()

Game -> MoveValidator : validateMove()

MoveValidator -> Token : checkState()

MoveValidator --> Game : valid

Game -> MoveExecutor : executeMove()

MoveExecutor -> Board : getCells()

MoveExecutor -> Token : updatePosition()

Board --> MoveExecutor : updated

MoveExecutor --> Game : success

Game -> Game : switchTurn()

Game --> Player : nextTurn
```

---

# 22. Activity Diagram

```text
Start
 |
Player turn
 |
Roll Dice
 |
Choose Token
 |
Is token at base?
 | YES
Dice == 6 ?
 | YES
Enter board
 | NO
Skip turn
 |
NO
 |
Move token
 |
Capture opponent token?
 | YES
Send opponent to base
 |
Check if token reached home
 |
Check win condition
 |
Switch turn
 |
End
```

---

# 23. System Flow

```text
Player Input
     |
     v
Game
     |
     v
DiceService
     |
     v
MoveValidator
     |
     v
MoveExecutor
     |
     v
Board
     |
     v
GameState Update
```

---

# 24. Interview Explanation (30-second answer)

You can say:

> The Game class orchestrates the entire Ludo gameplay. A player rolls the dice using DiceService, selects a token, and the Game creates a Move object. The MoveValidator checks if the move is valid according to Ludo rules. If valid, the MoveExecutor updates the board and token positions. After execution, the Game checks win conditions and switches the turn.

---

# 25. Senior Level Improvements (FAANG)

Add:

### Event System

Observer pattern for UI updates.

### Move History

Used for replay.

### Rule Engine

```text
CaptureRule
SafeCellRule
HomeEntryRule
DiceSixRule
```

### Multiplayer Support

GameManager manages multiple games.

---

# 26. Production Architecture

```text
Client
 |
Game API
 |
GameManager
 |
Game
 |
+ DiceService
+ MoveValidator
+ MoveExecutor
 |
Board
 |
Cells
 |
Tokens
```

---
