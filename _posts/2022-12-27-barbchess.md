---
date: 2022-12-27 00:00:00
layout: post
title: BarbChess - Building a chess engine
subtitle: My attempt to create my own chess engine.
image: /assets/img/barbchess_logo.png
description: >-
    Exploring chess programming through developing a chess engine.
category: games
---

> Try to beat the AI [here](https://barbiche.itch.io/barbchess)!

I've been playing chess recently, although I'm not very good at it. Platforms like [lichess](https://lichess.org/) or [chess.com](https://www.chess.com/home) really managed to turn one of the oldest game ever made into a modern competitive online game, with matchmaking, ranking, tools, learning contents, and also training AI, if the player does not want to face human players directly. __Chess.com__ in this regards is very interesting, as the user can face many AI that have specific traits in their play style.

These incredible chess platforms, plus the amazing Sebastian Lague's video on chess programming that has stayed in my mind for many months, motivated me to create as well __my own chess engine__, and on top of this, __a custom AI__ that I would be able to tweak around many play styles.

<iframe width="560" height="315" src="https://www.youtube.com/embed/U4ogK0MIzqk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

It took me some time to realize that creating a chess engine __stable__ enough to generate correctly all legal moves while being __performant__ to calculate these 3 or 4 turns in advance, is already quite challenging, so there won't be a lot of AI tweaking for now. Maybe another time!

# Structure

## BarbChessEngine
The whole code is divided in two repositories. One is the __engine__, named _BarbChessEngine_: it contains the board representation and the overall analysis on it. It has 3 main capabilities:

- Create a board from a FEN.

> FEN stands for _Forsyth-Edwards Notation_, and is a standard notation for describing a particular board position of a chess game. It contains not only all the piece positions, but also which color is expected to play, which castles are available, and also if an _en passant_ move is currently available or not.

![fen_to_board](/assets/img/barbchess/fen_to_board.png)

- Give all the legal moves playable for a board. Moves are exposed in the form of __FlaggedMove__, which is a structure that exposes not only the movement being done (start and end coordinate), but also additional information like if the move is a castle or not, if it is eating something, etc.

![generate_moves](/assets/img/barbchess/generate_moves.png)

- Execute a move, changing the board accordingly. This move should obviously be chosen among the previously generated __FlaggedMove__.

![generate_moves](/assets/img/barbchess/execute_move.png)

This project has been made in C#, and the source can be found [here](https://gitlab.com/Barbiche/barbchessengine).

## BarbChess

_BarbChess_ is a Unity application that leverages _BarbChessEngine_ by providing a render of the board. It also contains the implementation of different kind of __players__, which concretely are entities expected to choose a move to play in order to continue the game. Currently it handles:

- A human player, which chooses moves by actually moving pieces on the board.
- A random AI, which simply chooses a random move among the legal ones.
- _BarbAI_, the custom AI, which is suppose to be at least smarter than the random one.

_BarbChess_ source code can be found [here](https://gitlab.com/Barbiche/barbchess). It is also available on [itch.io](https://barbiche.itch.io/barbchess), as a game where the player faces the AI.

# Board representation

A good board representation is fundamental for a chess engine. For this aspect, I felt very in touch with the way Sebastian Lague's did it in his chess programming video.

A chess piece is represented by a single integer, and its binary representation contains the piece type in the weakest 3 digits, and the piece color with the strongest ones. If we want to represent no piece, then its a zero.

![generate_moves](/assets/img/barbchess/piece_representation.png)

Then, the board itself is nothing more than a 64 integer array. Each square can then be represented from two ways: a classic chess coordinate with a __file__ and a __rank__ is convenient from a user point of view, but mostly we can use the __index in the array__ to ease calculation. Thus, we can also create an __array of offsets__ which represents which number one should add in order to move to a specific direction.

![board_with_numbers](/assets/img/barbchess/board_with_numbers.png)

At this stage, we have enough elements to easily fill this board from a FEN, by writing a simple translator, and assigning each piece to its corresponding square.

# Making moves

The next piece of work is to be able to apply moves to the board. This is not really a complicated task: a chess move is really nothing more than moving a piece from a start position to an end position, removing from the board a piece which may already be here.

There are some things to consider though:
 - After each move, some information must be computed from the new board and the move that has been done.
    - _Castles_ possibilities should be reconsidered after a move implying a rook, or a king.
    - Pawns that moved 2 squares should be recognized, because they defined a _en-passant_ square which can be used for the next move.
 - Using these information, _castles_ and _en-passant_ require obviously more work than regular moves.
 
 But besides the possibility of __making__ a move, it will be very important later on to be able to __unmake__ a move. This combination __make/unmake__ must be perfectly stable, as it is a critical path for an AI board evaluation.

 > I first wanted to store a board state before each move, so that on unmake, I'll be able to simply retrieve the old board state, instead of doubling the logic which could easily lead to mistakes. Clearly it is much faster to simply apply some small changes in the board array rather than copying and storing board states among the game. On the other hand, my implementation of the board is stateful: if __an AI wish to explore the board after making a move__, __the caller is expected to unmake it__.

# Generate legal moves

This is where the troubles are. The idea is simple: considering a board, give __all moves that the player expected to play__ (the player who has the ___trait___) __can legally do.__ Because in chess, we have the moves that can be done computed from the specific rules of each pieces (those are called ___pseudo-legals___), but in a modern chess engine, moves that lead to a king being eaten on the next turn are __illegal__, and thus not playable. __Legal moves__ are then all moves a player can do without exposing his king to an immediate loss, and if no legal moves are available, then it's a ___mat___ (or a _pat_ but let's not consider it for now).

## Pseudo-legal moves

Generating pseudo-legal moves is pretty straightforward, you just need to apply the chess rules for each piece in the board. I won't go into much details here, but I wanted to highlight two cases that are amazingly handheld in Sebastian Lague's video (again, what a great work):

### Straight lines
Taking our __offset array__ back, to generate moves on __straight lines__ (which are important for the bishops, the rooks, and the queen which combine the two), we can simply start from the piece position, take the offsets we want, apply it to our start position, and loop until we find another piece.

![offset_array](/assets/img/barbchess/offset_array.png)

```
piece_position = input
start_direction = 0 // 0 for rook or queen, 4 for bishop
end _direction = 0 // 4 for rook, 7 for bishop or queen

from i = start_direction to end_direction do
    create_move_to(board[piece_position + offsets[i]])
```

You can consider computing in advance for all squares in the board how many squares exist for every direction. This way you don't have to manage board limits anymore.

### Precomputed knight moves

Knights are really easy and convenient: regardless the state of the board, a knight on a specific square always have the exact same set of pseudo-legal moves. That mean we can compute these moves in advance for all squares in the board! Easy work :)

## Legal moves by exploration

A very simple and straightforward way of isolating legal moves from pseudo-legals is simply to __play the move__. If the opponent has a pseudo-legal move that ends on the opposite king square, then you know the move you just did is not legal.

> From this board, trait to black:
>
> ![pseudo_0](/assets\img\barbchess\pseudo_2.png)
>
> Kd8, Kd7, Ke7, Kf8, Kf7 are the pseudo-legal moves. To compute legal moves, we play each of these moves, computing all opposite pseudo legal moves afterwards. We then realize that Kd8 leads to Rxd8, and Kd7 to Rxd7, so we remove Kd8 and Kd7 from the legal options.

This method has two great strengths: it is super __easy to implement__, and it is __very robust__, as long as your pseudo-legal move generation is correct. Every weird cases that can exist on a chess board are handheld with a very simple algorithm.

The major drawback is obviously the fact that it is __very inefficient__, as you need to generate all pseudo-legal moves for every possibilities. To cover our example board, this method requires 5 additional generations. With a more complex board it can easily reach 40.
However is it really an issue? If you go for a chess game with two human players, this solution is working well enough, so I think it is worthy to be mentioned. It will indeed be unusable with AI players that depends on the best performances possible for the move generation.

## Legal moves by analysis

The logic behind _BarbChessEngine_ legal move generation is represented with the following sequence:

![sequence](/assets\img\barbchess\sequence.png)

The first thing we want to do is to identify all squares attacked by opponent pieces. These are all the squares where, in any case, the king should be or should move to.

![attack_map](/assets/img/barbchess/attack_map.png)

Computing the previous attacking map, we are able to determine what are the __threats__ for the king, the pieces attacking him directly. If at least one threat exists, then we are in a __check__ situation.

There are two type of threats: __direct threats__ are pawns, knights and the king, and are easy to solve, because the next move won't change their respective capacities to threat the king. Pawns for example will always attack the same squares, whatever the movement done by its opponent.

![attack_map_1](/assets/img/barbchess/attack_map_1.png)

__Line threats__, so the rooks, bishops and the queen, are another story, as a chosen move can change the squares attacked by these threats. Pseudo-legal moves of a _pinned_ piece can expose the king to threats that yet didn't put the king in check.

![attack_map_2](/assets/img/barbchess/attack_map_2.png)

Line threats have to be considered, even if they don't directly have the king in target. I won't go through the process on how to recognize if a piece is a line threat or not, but what we need to know for every line threats is:
- The position of the threat
- If the threat is directly attacking the king or not
- A set of squares that are where the player could place a piece in order to __block__ the threat. Or on the contrary, where the player should not move a piece out of.

> In our previous example:
> h5 is a line threat.
> h5 is not attacking the king directly.
> Options to block the threat are b5, b5, d5, e5, f5, g5.

### Generate friendly pseudo-legal moves

This is the exact same step as we've done already, so I'll pass on this one.

### Prune illegal moves

Removing illegal moves from the pseudo-legals is the last step. It is easily done by using these rules:

- Remove all __king moves that ends in an attacked square__.
- Remove all __moves that start from the blocking squares of a line threat which do not end on blocking squares of the same line threat (or the line threat itself)__. In short, a pinned piece can move only on the attacking line of the piece which pinned it, or eat it.
- If the king is in __check__, then there is at least one attacker. Then a move is legal if:
    - it __finishes on the threat__.
    - it __finishes on a blocking option__ if the attacker is a line threat (except if it's the king).
    - the king is __moving out of attacking squares__.
- Finally, if the king is in __check__, but there is more than one attacker, then __the only viable moves are king moves leaving attacking squares__.

Plus: __Castles are not available if the king is in check!__

> The _en-passant_ rule troubles a bit these conditions: for example you can eat a piece that was blocking the line of a line threat without blocking it, which is pretty unique. So a bit of additional work is needed on this subject.

And this is it, we have our legal moves!

# Testing

To assess the robustness of the legal move generation, the [Chess Programming Wiki](https://www.chessprogramming.org/Perft) proposes a performance test (Perft) which count all legal moves walking in the generation tree, and comparing it to predetermined values.

For example, starting from the initial board, white has __20 legal moves__ (8 pawns that move 1 or 2 squares forward and 4 knight moves). If we make one of these moves, we can count how many moves black has, which is __20 as well__. Doing this for every white moves, __the sum of possibilities from the initial board, after a exploration depth of 2, is 400.__ Obviously you can go way deeper:

<table>
  <thead>
    <tr>
      <th>Depth</th>
      <th>Nodes</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>20</td>
    </tr>
    <tr>
      <td>2</td>
      <td>400</td>
    </tr>
    <tr>
      <td>3</td>
      <td>8,902</td>
    </tr>
    <tr>
      <td>4</td>
      <td>197,281</td>
    </tr>
    <tr>
      <td>5</td>
      <td>4,865,609</td>
    </tr>
    <tr>
      <td>6</td>
      <td>119,060,324</td>
    </tr>
  </tbody>
</table>

These precomputed results also exist for different positions, and are very valuable to debug the move generation.

For now, _BarbChessEngine_ don't go that far in depth for every tests, which means there is still room for improvement. There are still some very special cases that are very hard to see and catch, and I believe this engine is already capable enough to go to the next step, and develop an AI.

# _BarbAI_

As I said earlier, programming the engine was already a task, so for now, I've settled with a simple AI: I'm pretty sure though that it could be challenging enough for beginners!

_BarbAI_ is a __MiniMax implementation with Alpha-Beta pruning__. In short, it explores all possible moves, gives them a evaluation based on a evaluation function, and takes the move that minimizes the losses for the AI (supposing that the player will always take the better play). The two important points are:

- The evaluation function that rates a board. This first version is very simple, it counts the number of pieces for each player (each type of piece having its own value - a queen is 9x better than a pawn) and returns the difference. This means this version has no understanding of positioning in the board, the only thing that matters in the end are capture moves.

- The depth of the _MiniMax_ algorithm is crucial as well. It needs to be able to compute many moves in advance, otherwise it is very easy to beat. The ability to go deeper in a reasonable amount of time is directly linked to the operation that will be called the most in the move exploration: the legal move generation - that's why it is that important to have it quick!

# Conclusion

Programming a chess engine revealed itself to be harder than I thought, but it was a very interesting challenge. There will be a part 2 focusing on a smart AI, way harder to beat than this first version, but I'll have to leave it to the future. I hope you found it interesting though!

# References

- [Sebastians Lague's Chess Programming video](https://youtu.be/U4ogK0MIzqk), clearly the major reference.
- [Chess Programming Wiki](https://www.chessprogramming.org/Main_Page) that contains a tremendous amount of useful info.
- [Wikipedia on MiniMax](https://en.wikipedia.org/wiki/Minimax)


