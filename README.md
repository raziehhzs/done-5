import random
import argparse 
from typing import List, Optional, Tuple

Grid  List[List[int]]

def print_grid(grid: Grid) -> None:
    def cell(v): return "." if v == 0 else str(v)
    line = "+-------+-------+-------+"
    for r in range(9):
        if r % 3 == 0:
            print(line)
        row = []
        for c in range(9):
            if c % 3 == 0:
                row.append("| ")
            row.append(cell(grid[r][c]) + " ")
        row.append("|")
        print("".join(row))
    print(line)

def find_empty(grid: Grid) -> Optional[Tuple[int,int]]:
    for r in range(9):
        for c in range(9):
            if grid[r][c] == 0:
                return r, c
    return None

def is_valid(grid: Grid, r: int, c: int, val: int) -> bool:
    # Row/Col
    if any(grid[r][x] == val for x in range(9)): return False
    if any(grid[x][c] == val for x in range(9)): return False
    # Box
    br, bc = (r//3)*3, (c//3)*3
    for i in range(3):
        for j in range(3):
            if grid[br+i][bc+j] == val:
                return False
    return True

def solve(grid: Grid) -> bool:
    """Backtracking solver. Modifies grid in-place. Returns True if solved."""
    spot = find_empty(grid)
    if not spot:
        return True
    r, c = spot
    # Heuristic: shuffled 1..9
    for val in random.sample(range(1,10), 9):
        if is_valid(grid, r, c, val):
            grid[r][c] = val
            if solve(grid):
                return True
            grid[r][c] = 0
    return False

def solved_copy(grid: Grid) -> Grid:
    g = [row[:] for row in grid]
    if not solve(g):
        raise ValueError("این جدول حل‌پذیر نیست.")
    return g

def generate_full() -> Grid:
    """Generate a complete valid Sudoku grid."""
    grid = [[0]*9 for _ in range(9)]
    # Fill diagonal 3x3 boxes first (speeds up)
    for b in range(0,9,3):
        nums = random.sample(range(1,10), 9)
        k = 0
        for i in range(3):
            for j in range(3):
                grid[b+i][b+j] = nums[k]; k+=1
    # Solve to complete
    solve(grid)
    return grid

def count_solutions(grid: Grid, limit: int = 2) -> int:
    """Count up to 'limit' solutions (for uniqueness check)."""
    cnt = 0
    def dfs():
        nonlocal cnt
        if cnt >= limit: 
            return
        spot = find_empty(grid)
        if not spot:
            cnt += 1
            return
        r, c = spot
        for val in range(1,10):
            if is_valid(grid, r, c, val):
                grid[r][c] = val
                dfs()
                if cnt >= limit:
                    break
                grid[r][c] = 0
    dfs()
    return cnt

def remove_cells_to_difficulty(grid: Grid, difficulty: str) -> Grid:
    """Remove cells while keeping a unique solution, based on difficulty."""
    difficulty = difficulty.lower()
    targets = {
        "easy": 40,    # ~40 givens (51 removals)
        "medium": 32,  # ~32 givens
        "hard": 26     # ~26 givens
    }
    givens = targets.get(difficulty, 40)
    puzzle = [row[:] for row in grid]
    # Candidate positions to remove
    cells = [(r,c) for r in range(9) for c in range(9)]
    random.shuffle(cells)
    removed = 0
    # Aim for 81 - givens removals
    to_remove = 81 - givens
    for r, c in cells:
        if removed >= to_remove:
            break
        backup = puzzle[r][c]
        if backup == 0:
            continue
        puzzle[r][c] = 0
        # Check uniqueness cheaply
        tmp = [row[:] for row in puzzle]
        if count_solutions(tmp, limit=2) != 1:
            puzzle[r][c] = backup  # revert
        else:
            removed += 1
    return puzzle

def read_grid_from_file(path: str) -> Grid:
    """
    Read a 9x9 grid from file. Accepts digits and separators.
    Use 0 or . for empty cells.
    """
    rows: List[List[int]] = []
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            digits = [ch for ch in line if ch.isdigit() or ch == "."]
            row: List[int] = []
            for ch in digits:
                if ch == ".": ch = "0"
                row.append(int(ch))
                if len(row) == 9:
                    break
            if row:
                if len(row) != 9:
                    raise ValueError("هر سطر باید ۹ عدد داشته باشد.")
                rows.append(row)
            if len(rows) == 9:
                break
    if len(rows) != 9:
        raise ValueError("فایل باید ۹ سطر معتبر داشته باشد.")
    return rows

def is_complete_and_valid(grid: Grid) -> bool:
    # Check rows, cols, boxes contain 1..9 each once
    req = set(range(1,10))
    for i in range(9):
        if set(grid[i]) != req: return False
        if set(grid[r][i] for r in range(9)) != req: return False
    for br in range(0,9,3):
        for bc in range(0,9,3):
            box = [grid[br+i][bc+j] for i in range(3) for j in range(3)]
            if set(box) != req: return False
    return True

def main():
    parser = argparse.ArgumentParser(
        description="تولید و حل سودوکو (بدون کتابخانه‌ی خارجی).")
    parser.add_argument("--difficulty", "-d", choices=["easy","medium","hard"],
                        default="easy", help="درجه سختی جدول تولیدی")
    parser.add_argument("--solve", action="store_true",
                        help="حل کردن جدول (روی ورودی یا جدول تولیدی)")
    parser.add_argument("--from-file", type=str,
                        help="خواندن جدول از فایل (۹×۹، صفر یا . = خانه خالی)")
    args = parser.parse_args()

    if args.from_file:
        grid = read_grid_from_file(args.from_file)
    else:
        full = generate_full()
        grid = remove_cells_to_difficulty(full, args.difficulty)

    print("\n📋 جدول سودوکو:")
    print_grid(grid)

    if args.solve:
        g = [row[:] for row in grid]
        if solve(g):
            print("\n✅ حل جدول:")
            print_grid(g)
            if is_complete_and_valid(g):
                print("نتیجه معتبر است.")
            else:
                print("⚠️ حل به‌نظر نامعتبر می‌رسد.")
        else:
            print("\n❌ این جدول حل‌پذیر نیست.")

if __name__ == "__main__":
    random.seed()  # منبع تصادف سیستم
    main()
