# 鸡爪流高手

## 题目简述

游戏/协议逆向题。服务端低分保护判断的是“当前分数 < 10 且本次会扣分”，而不是结算后分数是否越界；在 ELO 公式和无符号分数字段下，可以把分数压到低位后输给更低分机器人，触发整数下溢并登顶排行榜。

## 解题过程

这道题设计的设计思路是：

1. 服务端希望实现“低分保护”，即当玩家分数已经很低时，不再继续扣分。

2. 这段保护的判断条件是“ 当前分数 < 10  且本次结算会扣分”，而不是“结算后分数不能低于

0”。

3. 因此只要玩家当前分数仍然大于等于 10 ，哪怕这一次失败会扣掉超过 10  分，更新逻辑仍然会

继续执行。

4. 在 ELO 公式下，当玩家分数处于 10 ~ 20  这样的低分区间时，如果去输给一个分数更低的机器

人，例如 0 分或 10 分机器人，扣分可能达到 10 、11 或 12 。

5. 一旦结算前分数例如是 10 、11 、12 ，而这次失败要扣 12  分，就会出现“更新前仍未触发

保护，更新后小于 0”的情况。

6. 如果分数字段使用无符号整型，那么这个负数结果会发生整数下溢，变成一个极大的正数，玩家会

瞬间登顶排行榜。

当前实现里，数据库会稳定生成一批关键低分机器人：

```text
50, 40, 30, 20, 20, 10
```

与此同时，默认 PLAY 不再按分差最近匹配，而是会从低分机器人池中随机抽取一名对手。因此预期解不再是“按一次固定顺序点名对局”，而是“不断等待目标对手出现，再执行对应的输赢动作”。

为了让这种等待不额外干扰分数，当前实现还额外提供了一个规则：

- 如果开局后棋盘仍为空，玩家立刻 RESIGN

- 这次对局会被判定为平局

- 双方分数都不会变化

因此，预期利用步骤应理解为下面这条“等待式”路径：

1. 不断 PLAY

2. 如果当前对手不是目标分数点，就在开局直接 RESIGN ，以平局结束并继续等待

3. 当随机到 50  分机器人时，先落一手再认输

4. 之后依次对 40 、30 、20  分机器人重复这个过程，把自己的分数压到 10

5. 等到 10  分机器人出现时，真正下棋获胜，使自己回到 20 ，同时把该机器人打到 0

6. 再等待另一个 20  分机器人出现，先落一手再认输，让自己回到 10

7. 最后等待刚刚造出来的 0  分机器人出现，先落一手再认输，触发整数下溢

### Exploit

common.py

```python
import os
import re
import sqlite3
import math
import sys
from pathlib import Path
from pwn import context, p8, p32, process, remote
MAGIC = b"GAME"
GET_FLAG = 1
GET_SCORE = 2
GET_RANK = 3
RESET = 4
PLAY = 5
GET_BOARD = 6
MOVE = 7
RESIGN = 8
ROOT = Path(__file__).resolve().parent.parent
DB_PATH = ROOT / "game.db"
DEFAULT_FLAG = "flag{demo_full_exp_success}"
ATTACK_CHAIN = [50, 40, 30, 20]
WIN_SEQUENCE = [(7, 7), (7, 6), (7, 8), (7, 5), (7, 9)]
context.log_level = "error"
def find_server_path() -> Path:
    candidates = [
        ROOT / "build" / "Release" / "server.exe",
        ROOT / "build" / "server.exe",
        ROOT / "build" / "server",
    ]
    for path in candidates:
        if path.exists():
            return path
    raise FileNotFoundError("server executable not found")
def build_packet(msg_type: int, param: bytes = b"") -> bytes:
    return MAGIC + p32(len(param) + 9, endian="big") + p8(msg_type) + param
def parse_score_rank(message: str) -> tuple[int, int]:
    score_match = re.search(r"Score=(\d+)", message)
    rank_match = re.search(r"Rank=(\d+)", message)
    if score_match is None or rank_match is None:
        raise ValueError(f"cannot parse score/rank from {message!r}")
    return int(score_match.group(1)), int(rank_match.group(1))
def parse_matched_opponent(message: str) -> tuple[str, int]:
    match = re.search(r"^Matched (.+) Score=(\d+) You=Black$", message)
    if match is None:
        raise ValueError(f"cannot parse opponent from {message!r}")
    return match.group(1), int(match.group(2))
def format_board_message(message: str) -> str:
    if "Board=" not in message:
        return message
    prefix, board = message.split("Board=", 1)
    rows = board.split("/")
    return f"{prefix}Board=\n" + "\n".join(rows)
def calc_delta(player_score: int, opponent_score: int, result: float) -> int:
    exponent = (opponent_score - player_score) / 50.0
    if exponent > 308:
        expected = 0.0
    elif exponent < -308:
        expected = 1.0
    else:
        expected = 1.0 / (1.0 + math.pow(10.0, exponent))
    return round(20 * (result - expected))
def apply_buggy_update(player_score: int, opponent_score: int, result: float) -
> tuple[int, int, bool]:
    delta = calc_delta(player_score, opponent_score, result)
    if player_score < 10 and delta < 0:
        return player_score, delta, False
    wrapped = player_score + delta < 0
    next_score = (player_score + (delta % (2**32))) % (2**32)
    return next_score, delta, wrapped
def find_bot_name_by_score(score: int) -> str:
    conn = sqlite3.connect(DB_PATH)
    try:
        cursor = conn.execute(
            "SELECT name FROM players WHERE name != 'player' AND score = ?
ORDER BY id ASC LIMIT 1",
            (score,),
        )
        row = cursor.fetchone()
        if row is None:
            raise ValueError(f"no bot found with score {score}")
        return str(row[0])
    finally:
        conn.close()
class ServerClient:
    def __init__(self, server_path: Path | None = None, flag: str =
"flag{local_test_flag}"):
        self.server_path = server_path
        self.remote_host = os.environ.get("CHICKEN_FEET_HOST")
        self.remote_port = int(os.environ.get("CHICKEN_FEET_PORT", "0"))
        if self.remote_host:
            if self.remote_port <= 0:
                raise ValueError("CHICKEN_FEET_PORT must be set to a positive
integer when CHICKEN_FEET_HOST is used")
            self.process = remote(self.remote_host, self.remote_port)
        else:
            self.server_path = self.server_path or find_server_path()
            self.process = process(
                [str(self.server_path)],
                cwd=str(ROOT),
                env={**os.environ, "FLAG": flag, "GZCTF_FLAG": flag},
            )
    def close(self) -> None:
        poll = getattr(self.process, "poll", None)
        kill = getattr(self.process, "kill", None)
        if callable(poll) and callable(kill):
            if self.process.poll(block=False) is None:
                self.process.kill()
        self.process.close()
    def _read_exact(self, size: int) -> bytes:
        data = self.process.recvn(size, timeout=5)
        if len(data) != size:
            raise EOFError(f"expected {size} bytes, got {len(data)}")
        return data
    def request(self, msg_type: int, param: bytes = b"") -> str:
        self.process.send(build_packet(msg_type, param))
        magic = self._read_exact(4)
        if magic != MAGIC:
            raise ValueError(f"bad response magic: {magic!r}")
        total_length = int.from_bytes(self._read_exact(4), "big")
        _response_type = self._read_exact(1)[0]
        payload = self._read_exact(total_length - 9)
        return payload.decode("utf-8", errors="replace")
    def get_score(self) -> str:
        return self.request(GET_SCORE)
    def get_rank(self) -> str:
        return self.request(GET_RANK)
    def reset(self) -> str:
        return self.request(RESET)
    def play(self, opponent_name: str | None = None) -> str:
        param = b"" if opponent_name is None else opponent_name.encode()
        return self.request(PLAY, param)
    def get_board(self) -> str:
        return self.request(GET_BOARD)
    def move(self, x: int, y: int) -> str:
        return self.request(MOVE, f"{x},{y}".encode())
    def resign(self) -> str:
        return self.request(RESIGN)
    def get_flag(self) -> str:
        return self.request(GET_FLAG)
def print_step(title: str, message: str) -> None:
    print(f"[{title}] {format_board_message(message)}")
def wait_for_random_opponent(
    client: "ServerClient",
    predicate,
    target_desc: str,
    max_attempts: int = 2000,
) -> tuple[str, int, str]:
    attempt = 0
    while attempt < max_attempts:
        attempt += 1
        play_message = client.play()
        name, score = parse_matched_opponent(play_message)
        print_step("PLAY", f"attempt={attempt} {play_message}")
        if predicate(name, score):
            return name, score, play_message
        resign_message = client.resign()
        print_step("RESIGN", resign_message)
        if not resign_message.startswith("Draw "):
            fail(f"expected opening resign to be Draw while waiting for
{target_desc}, got {resign_message}")
    fail(f"did not find target opponent: {target_desc} within {max_attempts}
attempts")
    raise AssertionError("unreachable")
def find_bot_name_by_score_excluding(score: int, excluded_names: set[str]) ->
str:
    conn = sqlite3.connect(DB_PATH)
    try:
        placeholders = ",".join("?" for _ in excluded_names)
        if excluded_names:
            query = (
                f"SELECT name FROM players WHERE name != 'player' AND score =
? "
                f"AND name NOT IN ({placeholders}) ORDER BY id ASC LIMIT 1"
            )
            params = (score, *sorted(excluded_names))
        else:
            query = "SELECT name FROM players WHERE name != 'player' AND score
= ? ORDER BY id ASC LIMIT 1"
            params = (score,)
        cursor = conn.execute(query, params)
        row = cursor.fetchone()
        if row is None:
            raise ValueError(f"no bot found with score {score} excluding
{sorted(excluded_names)}")
        return str(row[0])
    finally:
        conn.close()
def fail(message: str) -> None:
    print(f"[FAIL] {message}", file=sys.stderr)
    raise SystemExit(1)
def expected_flag(default: str = DEFAULT_FLAG) -> str:
    return os.environ.get("CHICKEN_FEET_EXPECTED_FLAG", default)
def expect_prefix(message: str, prefixes: tuple[str, ...], context: str) ->
None:
    if not any(message.startswith(prefix) for prefix in prefixes):
        fail(f"{context}: {message}")
def reset_and_check(client: "ServerClient", expected_score: int = 50) ->
tuple[int, int]:
    reset_message = client.reset()
    print_step("RESET", reset_message)
    score, rank = parse_score_rank(reset_message)
    if score != expected_score:
        fail(f"expected reset score {expected_score}, got {score}")
    return score, rank
def lose_to_score(client: "ServerClient", target_score: int, excluded_names:
set[str] | None = None) -> tuple[str, str]:
    excluded = excluded_names or set()
    bot_name, _bot_score, _play_message = wait_for_random_opponent(
        client,
        lambda name, score: score == target_score and name not in excluded,
        f"score={target_score}",
    )
    move_message = client.move(0, 0)
    print_step("MOVE", move_message)
    expect_prefix(
        move_message,
        ("Continue ",),
        f"expected a normal first move before resigning against score
{target_score}",
    )
    result = client.resign()
    print_step("RESIGN", result)
    expect_prefix(result, ("Resign ",), f"unexpected settlement after
resigning vs score {target_score}")
    return bot_name, result
def win_against_score_10(client: "ServerClient") -> tuple[str, str]:
    bot_name, _bot_score, _play_message = wait_for_random_opponent(
        client,
        lambda _name, score: score == 10,
        "score=10",
    )
    for x, y in WIN_SEQUENCE:
        response = client.move(x, y)
        print_step("MOVE", response)
        if response.startswith("Win "):
            return bot_name, response
        expect_prefix(response, ("Continue ",), "unexpected move response
while forcing win")
    fail("winning sequence did not finish with a win")
    raise AssertionError("unreachable")
def run_attack_chain(client: "ServerClient") -> tuple[int, int]:
    for target_score in ATTACK_CHAIN:
        bot_name, result = lose_to_score(client, target_score)
        score, rank = parse_score_rank(result)
        print_step("STATE", f"after losing to {bot_name}: score={score} rank=
{rank}")
    zero_bot_name, win_message = win_against_score_10(client)
    score_after_win, rank_after_win = parse_score_rank(win_message)
    print_step("STATE", f"after beating the 10-score bot: score=
{score_after_win} rank={rank_after_win}")
    lose_to_score(client, 20, {zero_bot_name})
    _bot_name, final_message = lose_to_score(client, 0)
    wrapped_score, wrapped_rank = parse_score_rank(final_message)
    print_step("WRAP", final_message)
    return wrapped_score, wrapped_rank
```

```text
from common import (
    expected_flag,
    fail,
    parse_score_rank,
    print_step,
    reset_and_check,
    run_attack_chain,
    ServerClient,
)
def main() -> None:
    client = ServerClient(flag=expected_flag())
    try:
        reset_and_check(client)
        run_attack_chain(client)
        score_message = client.get_score()
        print_step("GET_SCORE", score_message)
        score, rank = parse_score_rank(score_message)
        print_step("STATE", f"post-chain score={score} rank={rank}")
        if rank != 1:
            fail(f"exploit chain finished without rank 1, got rank {rank}")
        flag_message = client.get_flag()
        print_step("GET_FLAG", flag_message)
        if flag_message != expected_flag():
            fail(f"unexpected flag response: {flag_message}")
        print_step("RESULT", "full exploit succeeded")
    finally:
        client.close()
if __name__ == "__main__":
    main()
```

## 方法总结

- 核心技巧：ELO 分数结算边界错误导致无符号整数下溢。
- 识别信号：低分保护只检查更新前状态，且结算结果写入无符号字段时，要检查扣分跨过 0 的下溢。
- 复用要点：利用随机匹配时可用空棋盘 RESIGN 等无损操作等待目标对手，再按固定输赢链塑造分数。
