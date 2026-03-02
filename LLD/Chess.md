
Chess LLD

For a **7+ years interview**, a Chess LLD should demonstrate:

* Clean **domain modeling**
* **Clear responsibility separation**
* Use of **design patterns**
* **Move validation rules**
* **Game orchestration**
* **Sequence flow**
* **Extensibility**

I'll design a **proper production-style Chess LLD**.

---

# 1. Requirements of Chess System

### Functional

* Two players
* 8x8 board
* Pieces move based on rules
* Capture pieces
* Check
* Checkmate
* Turn based
* Move history

### Non-Functional

* Extensible
* Maintainable
* Rule engine separation

---

# 2. Core Design Patterns Used

| Pattern       | Usage                                  |
| ------------- | -------------------------------------- |
| **Strategy**  | Different piece movement rules         |
| **Factory**   | Create chess pieces                    |
| **Command**   | Represent a move                       |
| **State**     | Game states (ACTIVE, CHECK, CHECKMATE) |
| **Observer**  | UI updates                             |
| **Singleton** | Game manager                           |

---

# 3. High Level Architecture

```
Player
   |
   v
Game
   |
   v
MoveExecutor
   |
   v
MoveValidator
   |
   v
Board
   |
   v
Piece
   |
   v
MoveStrategy
```

---

# 4. Core Entities

```
Game
Board
Cell
Player
Piece (abstract)

Pawn
Rook
Knight
Bishop
Queen
King

Move
MoveValidator
MoveExecutor
MoveStrategy
```

---

# 5. Project Structure

```
chess
 ├── model
 │      Game
 │      Board
 │      Cell
 │      Move
 │      Player
 │
 ├── piece
 │      Piece
 │      Pawn
 │      Rook
 │      Bishop
 │      Knight
 │      Queen
 │      King
 │
 ├── strategy
 │      MoveStrategy
 │      PawnMoveStrategy
 │      RookMoveStrategy
 │      KnightMoveStrategy
 │
 ├── service
 │      MoveValidator
 │      MoveExecutor
 │
 ├── factory
 │      PieceFactory
 │
 └── enums
        Color
        GameState
```

---

# 6. Core Models

## Player

```
class Player {

    private String id;
    private Color color;

}
```

---

# 7. Move Class (Command Pattern)

Represents a **player action**.

```
class Move {

    private Player player;

    private Position from;
    private Position to;

    private Piece movedPiece;
    private Piece capturedPiece;

}
```

---

# 8. Board

```
class Board {

    private Cell[][] grid = new Cell[8][8];

    public Cell getCell(Position pos){
        return grid[pos.row][pos.col];
    }

}
```

---

# 9. Cell

```
class Cell {

    private Position position;
    private Piece piece;

}
```

---

# 10. Piece (Abstract)

Using **Strategy Pattern**

```
abstract class Piece {

    protected Color color;

    protected MoveStrategy strategy;

    public List<Position> validMoves(Board board, Position position){
        return strategy.generateMoves(board, position);
    }

}
```

---

# 11. Concrete Piece Example

```
class Rook extends Piece {

    public Rook(Color color){
        this.color = color;
        this.strategy = new RookMoveStrategy();
    }

}
```

---

# 12. MoveStrategy Interface

```
interface MoveStrategy {

    List<Position> generateMoves(Board board, Position from);

}
```

---

# 13. Example Strategy

### RookMoveStrategy

```
class RookMoveStrategy implements MoveStrategy {

    public List<Position> generateMoves(Board board, Position from){

        List<Position> moves = new ArrayList<>();

        // horizontal and vertical traversal

        return moves;
    }

}
```

---

# 14. PieceFactory (Factory Pattern)

```
class PieceFactory {

    public static Piece createPiece(String type, Color color){

        switch(type){

            case "PAWN":
                return new Pawn(color);

            case "ROOK":
                return new Rook(color);

            case "KNIGHT":
                return new Knight(color);

            case "BISHOP":
                return new Bishop(color);

            case "QUEEN":
                return new Queen(color);

            case "KING":
                return new King(color);
        }

        return null;
    }

}
```

---

# 15. MoveValidator

Responsible for **checking if move is legal**.

```
class MoveValidator {

    public boolean validateMove(Board board, Move move){

        Piece piece = move.getMovedPiece();

        List<Position> validMoves =
                piece.validMoves(board, move.getFrom());

        return validMoves.contains(move.getTo());

    }

}
```

---

# 16. MoveExecutor

Responsible for **executing move on board**.

```
class MoveExecutor {

    public void executeMove(Board board, Move move){

        Cell from = board.getCell(move.getFrom());
        Cell to = board.getCell(move.getTo());

        move.setCapturedPiece(to.getPiece());

        to.setPiece(from.getPiece());

        from.setPiece(null);

    }

}
```

---

# 17. Game Class (Orchestrator)

The **Game class controls the whole flow**.

```
class Game {

    private Board board;

    private Player whitePlayer;
    private Player blackPlayer;

    private MoveValidator validator;
    private MoveExecutor executor;

    private Player currentPlayer;

}
```

---

# 18. Game Flow

```
public void playMove(Position from, Position to){

    Cell cell = board.getCell(from);

    Piece piece = cell.getPiece();

    Move move = new Move(currentPlayer, from, to, piece);

    if(!validator.validateMove(board, move)){
        throw new RuntimeException("Invalid move");
    }

    executor.executeMove(board, move);

    switchTurn();
}
```

---

# 19. Sequence Diagram (Move Flow)

```
Player
   |
   v
Game.playMove()
   |
   v
Move created
   |
   v
MoveValidator.validate()
   |
   v
Piece.validMoves()
   |
   v
MoveStrategy.generateMoves()
   |
   v
MoveExecutor.execute()
   |
   v
Board updated
   |
   v
Game.switchTurn()
```

---

# 20. Sequence Diagram (Text Form)

```
Player -> Game : playMove()
Game -> MoveValidator : validate(move)
MoveValidator -> Piece : validMoves()
Piece -> MoveStrategy : generateMoves()

MoveValidator -> Game : valid

Game -> MoveExecutor : execute(move)
MoveExecutor -> Board : update cells
Game -> Game : switchTurn()
```

---

# 21. Chess Board Initialization

```
public void initializeBoard(){

    board.place(new Rook(Color.WHITE), 0,0);
    board.place(new Knight(Color.WHITE), 0,1);
    board.place(new Bishop(Color.WHITE), 0,2);
    board.place(new Queen(Color.WHITE), 0,3);
    board.place(new King(Color.WHITE), 0,4);

}
```

---

# 22. Game State Pattern

```
enum GameState {

    ACTIVE,
    CHECK,
    CHECKMATE,
    STALEMATE

}
```

---

# 23. Important Interview Discussion Points

### Why Strategy Pattern?

Each piece has **different movement logic**.

Example

```
Pawn → forward + diagonal capture
Rook → straight
Knight → L shape
```

Encapsulating this prevents huge `if-else`.

---

### Why Command Pattern (Move)?

Move allows:

* Undo
* Replay
* Logging
* Multiplayer sync

---

### Why Factory?

Creating pieces in multiple places is messy.

Factory centralizes creation.

---

# 24. Extension Features

You can add:

### Castling

### En passant

### Promotion

### Check detection

### AI opponent

---

# 25. Most Important Interview Insight (Senior Level)

Avoid doing:

```
if(piece == ROOK)
if(piece == KNIGHT)
if(piece == PAWN)
```

Instead use **polymorphism + strategy**.

Interviewers love this.

---

# 26. Final Architecture

```
           Player
              |
              v
            Game
              |
   -----------------------
   |                     |
MoveValidator        MoveExecutor
   |                     |
   v                     v
 Piece               Board
   |
   v
MoveStrategy
```


-------------------------------------------------------------------------

Below is a **complete interview-level Chess LLD implementation (condensed but realistic)** suitable for **7+ years experience**.
It shows **who instantiates what, how objects interact, patterns used, and core move logic**.

Design patterns used:

* **Strategy** → Piece movement
* **Factory** → Piece creation
* **Command** → Move object
* **State** → Game state
* **Orchestrator (Game)** → Controls flow

---

# 1. Project Structure

```
chess
 ├── model
 │      Game
 │      Board
 │      Cell
 │      Position
 │      Move
 │      Player
 │
 ├── piece
 │      Piece (abstract)
 │      Pawn
 │      Rook
 │      Knight
 │      Bishop
 │      Queen
 │      King
 │
 ├── strategy
 │      MoveStrategy
 │      PawnMoveStrategy
 │      RookMoveStrategy
 │      BishopMoveStrategy
 │      KnightMoveStrategy
 │      QueenMoveStrategy
 │      KingMoveStrategy
 │
 ├── service
 │      MoveValidator
 │      MoveExecutor
 │
 ├── factory
 │      PieceFactory
 │
 └── enums
        Color
        GameState
```

---

# 2. Enums

```java
enum Color {
    WHITE,
    BLACK
}

enum GameState {
    ACTIVE,
    CHECK,
    CHECKMATE,
    STALEMATE
}
```

---

# 3. Position

```java
class Position {

    int row;
    int col;

    Position(int r, int c){
        row = r;
        col = c;
    }

    boolean isValid(){
        return row >=0 && row < 8 && col >=0 && col < 8;
    }
}
```

---

# 4. Cell

```java
class Cell {

    Position position;
    Piece piece;

    Cell(Position pos){
        this.position = pos;
    }

    boolean isEmpty(){
        return piece == null;
    }
}
```

---

# 5. Board

```java
class Board {

    private Cell[][] grid = new Cell[8][8];

    Board(){
        initialize();
    }

    private void initialize(){

        for(int r=0;r<8;r++){
            for(int c=0;c<8;c++){
                grid[r][c] = new Cell(new Position(r,c));
            }
        }
    }

    Cell getCell(Position pos){
        return grid[pos.row][pos.col];
    }

    void placePiece(Piece piece, int row, int col){
        grid[row][col].piece = piece;
    }
}
```

---

# 6. Player

```java
class Player {

    String name;
    Color color;

    Player(String name, Color color){
        this.name = name;
        this.color = color;
    }
}
```

---

# 7. Move (Command Pattern)

```java
class Move {

    Player player;

    Position from;
    Position to;

    Piece movedPiece;
    Piece capturedPiece;

    Move(Player p, Position f, Position t, Piece piece){
        player = p;
        from = f;
        to = t;
        movedPiece = piece;
    }
}
```

---

# 8. Piece (Strategy Pattern)

```java
abstract class Piece {

    Color color;

    MoveStrategy strategy;

    Piece(Color color){
        this.color = color;
    }

    List<Position> validMoves(Board board, Position from){
        return strategy.generateMoves(board, from);
    }
}
```

---

# 9. Piece Implementations

Example:

### Rook

```java
class Rook extends Piece {

    Rook(Color color){
        super(color);
        this.strategy = new RookMoveStrategy();
    }
}
```

Same pattern used for:

```
Pawn
Knight
Bishop
Queen
King
```

---

# 10. MoveStrategy Interface

```java
interface MoveStrategy {

    List<Position> generateMoves(Board board, Position from);

}
```

---

# 11. Rook Move Strategy

```java
class RookMoveStrategy implements MoveStrategy {

    private static final int[][] DIRS = {
        {1,0},{-1,0},{0,1},{0,-1}
    };

    public List<Position> generateMoves(Board board, Position from){

        List<Position> moves = new ArrayList<>();

        for(int[] d : DIRS){

            int r = from.row + d[0];
            int c = from.col + d[1];

            while(r>=0 && r<8 && c>=0 && c<8){

                Position pos = new Position(r,c);
                Cell cell = board.getCell(pos);

                if(cell.isEmpty()){
                    moves.add(pos);
                }
                else{
                    moves.add(pos);
                    break;
                }

                r += d[0];
                c += d[1];
            }
        }

        return moves;
    }
}
```

---

# 12. Knight Move Strategy

```java
class KnightMoveStrategy implements MoveStrategy {

    int[][] MOVES = {
        {2,1},{2,-1},{-2,1},{-2,-1},
        {1,2},{1,-2},{-1,2},{-1,-2}
    };

    public List<Position> generateMoves(Board board, Position from){

        List<Position> moves = new ArrayList<>();

        for(int[] m : MOVES){

            Position p = new Position(from.row+m[0], from.col+m[1]);

            if(p.isValid()){
                moves.add(p);
            }
        }

        return moves;
    }
}
```

---

# 13. Bishop Strategy

```java
class BishopMoveStrategy implements MoveStrategy {

    int[][] DIRS = {
        {1,1},{1,-1},{-1,1},{-1,-1}
    };

    public List<Position> generateMoves(Board board, Position from){

        List<Position> moves = new ArrayList<>();

        for(int[] d : DIRS){

            int r = from.row+d[0];
            int c = from.col+d[1];

            while(r>=0 && r<8 && c>=0 && c<8){

                Position p = new Position(r,c);
                moves.add(p);

                if(!board.getCell(p).isEmpty())
                    break;

                r += d[0];
                c += d[1];
            }
        }

        return moves;
    }
}
```

---

# 14. MoveValidator

```java
class MoveValidator {

    boolean validate(Board board, Move move){

        Piece piece = move.movedPiece;

        List<Position> validMoves =
                piece.validMoves(board, move.from);

        for(Position p : validMoves){

            if(p.row == move.to.row &&
               p.col == move.to.col){
                return true;
            }
        }

        return false;
    }
}
```

---

# 15. MoveExecutor

```java
class MoveExecutor {

    void execute(Board board, Move move){

        Cell from = board.getCell(move.from);
        Cell to = board.getCell(move.to);

        move.capturedPiece = to.piece;

        to.piece = from.piece;
        from.piece = null;
    }
}
```

---

# 16. PieceFactory

```java
class PieceFactory {

    static Piece createPiece(String type, Color color){

        switch(type){

            case "ROOK": return new Rook(color);
            case "KNIGHT": return new Knight(color);
            case "BISHOP": return new Bishop(color);
            case "QUEEN": return new Queen(color);
            case "KING": return new King(color);
            case "PAWN": return new Pawn(color);

        }

        return null;
    }
}
```

---

# 17. Game (Main Orchestrator)

```java
class Game {

    Board board;

    Player white;
    Player black;

    Player currentPlayer;

    MoveValidator validator;
    MoveExecutor executor;

    Game(Player p1, Player p2){

        board = new Board();

        white = p1;
        black = p2;

        currentPlayer = white;

        validator = new MoveValidator();
        executor = new MoveExecutor();

        initializeBoard();
    }
```

---

# 18. Board Initialization

```java
void initializeBoard(){

    board.placePiece(new Rook(Color.WHITE),0,0);
    board.placePiece(new Knight(Color.WHITE),0,1);
    board.placePiece(new Bishop(Color.WHITE),0,2);
    board.placePiece(new Queen(Color.WHITE),0,3);
    board.placePiece(new King(Color.WHITE),0,4);

}
```

---

# 19. Game Move Flow

```java
public void playMove(Position from, Position to){

    Cell cell = board.getCell(from);

    Piece piece = cell.piece;

    Move move = new Move(currentPlayer, from, to, piece);

    if(!validator.validate(board, move)){
        throw new RuntimeException("Invalid move");
    }

    executor.execute(board, move);

    switchTurn();
}
```

---

# 20. Turn Switching

```java
void switchTurn(){

    if(currentPlayer == white)
        currentPlayer = black;
    else
        currentPlayer = white;
}
```

---

# 21. Sequence Flow

```
Player
  |
  v
Game.playMove()
  |
  v
Move created
  |
  v
MoveValidator.validate()
  |
  v
Piece.validMoves()
  |
  v
MoveStrategy.generateMoves()
  |
  v
MoveExecutor.execute()
  |
  v
Board updated
  |
  v
Game.switchTurn()
```

---

# 22. Why This Design is Strong

This design provides:

* **Extensibility**
* **Clean separation of responsibilities**
* **No giant if-else for pieces**
* **Reusable move engine**
* **Easy rule extensions**

---

# 23. Advanced Features (Add Later)

For a **senior-level extension**, you can add:

* Check detection engine
* Castling logic
* Pawn promotion
* En passant
* Undo move stack
* AI engine
