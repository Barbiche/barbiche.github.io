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



