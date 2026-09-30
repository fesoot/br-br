rows, colms = 5, 5
matrix = [["*" for _ in range(rows)] for _ in range(colms)]

for row in matrix:
    row[2] = "0"

for row in matrix:
    for item in row:
        print(item, end=" ")
    print()
