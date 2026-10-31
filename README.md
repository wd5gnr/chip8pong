# CHIP-8 Pong (Octo)

A two-player Pong for CHIP-8, written in [Octo](https://github.com/JohnEarnest/Octo) assembly. It compiles to **332 bytes**.

Features:

- Two paddles on a 64x32 screen, controlled from the hex keypad
- Ball angle depends on where it strikes the paddle
- Score display using the built-in hex font
- Sound on hits and misses
- First player to 9 wins; the game then restarts automatically

## Running it

**In the browser:** open the Octo IDE, paste the contents of `pong.8o` into the editor and press Run.

**From the command line:** with the Octo repository checked out,

```
./octo pong.8o pong.ch8
```

produces a ROM you can load in any CHIP-8 interpreter that has no special quirk requirements.

## Controls

The program reads the CHIP-8 hex keypad. On Octo's default QWERTY mapping:

| Player | Up | Down |
|--------|----|------|
| Left   | hex `1` (keyboard `1`) | hex `4` (keyboard `Q`) |
| Right  | hex `C` (keyboard `4`) | hex `D` (keyboard `R`) |

Paddles move one pixel per frame while the key is held.

## Rules

- The ball serves from the center. After a point, it is served toward the player who missed, with a random vertical direction.
- Missing the ball scores a point for the opponent.
- The first player to reach 9 wins. The final score and the paddles stay on screen for about 3 seconds, then both scores reset and a new game begins.

### Bounce angles

The vertical speed of the ball is set by where it strikes the paddle (offset 0 is the top pixel, 4 is the bottom):

| Offset | Result | Vertical step |
|--------|--------|---------------|
| 0 | up, steep | one row per 6 horizontal steps (about 10 degrees) |
| 1 | up, shallow | one row per 11 horizontal steps (about 5 degrees) |
| 2 | straight | no vertical motion |
| 3 | down, shallow | one row per 11 horizontal steps |
| 4 | down, steep | one row per 6 horizontal steps |

The ball always moves one pixel horizontally per frame.

## How it works

**Drawing.** Everything is drawn with XOR sprites. Each frame begins by drawing the ball and paddles again at their old positions, which erases them. They are then moved and drawn at their new positions.

**Collision.** The paddles are drawn first and the ball last, so the collision flag (`VF`) after drawing the ball means the ball overlaps a paddle. The flag is copied into a register immediately because comparisons overwrite `VF`. A hit is accepted only when the ball is at a paddle column and moving toward that paddle, so a ball cannot hit the same paddle twice.

**Bounce.** On a hit, the ball is drawn once more at the contact pixel, which restores the paddle pixel that the ball had toggled off. The ball is then stepped one pixel away from the paddle and redrawn. This avoids a one-frame notch in the paddle.

**Misses.** The ball is not on screen between the erase pass and the draw pass, so scores can be changed safely when a miss is detected.

**Timing.** Each frame ends by waiting for the delay timer (`FRAME_DELAY`, 2 ticks, about 30 frames per second). The game logic itself takes several timer ticks to execute at Octo's default tickrate, so the actual ball speed depends on the tickrate setting in Octo's options. If the game feels slow, raise the tickrate or set `FRAME_DELAY` to 1.

## Registers

| Register | Alias | Purpose |
|----------|-------|---------|
| `v0` | `left_y` | Top row of the left paddle |
| `v1` | `right_y` | Top row of the right paddle |
| `v2` | `ball_x` | Ball column |
| `v3` | `ball_y` | Ball row |
| `v4` | `dx` | Horizontal direction: `RIGHT` (1) or `LEFT` (255) |
| `v5` | `dy` | Vertical direction: `DOWN` (1) or `UP` (255) |
| `v6` | `offset` | Hit offset on the paddle (0 to 4); also used as a scratch row for the score digits |
| `v7` | `tmp` | Scratch (sprite x, key codes, timer values) |
| `v8` | `period` | Frames per vertical step (0 means no vertical motion) |
| `v9` | `count` | Frames remaining until the next vertical step |
| `vA` | `hit` | Copy of `VF` after the ball is drawn |
| `vC` | `score_l` | Left player's score |
| `vD` | `score_r` | Right player's score |

`vB` and `vE` are unused.

## Tuning

All the numbers you are likely to want to change are constants near the top of the file:

| Constant | Default | Meaning |
|----------|---------|---------|
| `WIN_SCORE` | 9 | Points needed to win |
| `GAME_OVER_PAUSE` | 180 | Pause after a win, in 60 Hz ticks |
| `FRAME_DELAY` | 2 | Timer ticks per frame |
| `STEEP`, `SHALLOW`, `FLAT` | 6, 11, 0 | Frames per vertical step for each bounce type |
| `HIT_BUZZ`, `MISS_BUZZ` | 3, 15 | Sound lengths, in 60 Hz ticks |
| `PADDLE_H` | 5 | Paddle height (also change the `paddle` sprite data) |
| `PADDLE_L_X`, `PADDLE_R_X` | 2, 61 | Paddle columns |
| `SERVE_X`, `SERVE_Y` | 32, 16 | Serve position |
| `SCORE_L_X`, `SCORE_R_X`, `SCORE_Y` | 26, 34, 1 | Score digit positions |

## Notes

- The ball can briefly erase pixels from a score digit while passing over it. The digit is restored as soon as the ball moves on.
- Scores are single digits because the game ends at 9; raising `WIN_SCORE` above 9 would need a second digit.
- If you move the paddles or change the hit columns, keep the ball's hit test columns equal to the paddle columns. If you ever make the ball move more than one pixel per frame, the hit test must also cover the pixels it skips.

