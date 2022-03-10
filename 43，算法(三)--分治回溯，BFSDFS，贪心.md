### 分治与回溯

terminator->process->drill down->reverse state

#### [求Pow(x,n)](https://leetcode-cn.com/problems/powx-n/)

```go
func myPow(x float64, n int) float64 {
    if n < 0 {
        return float64(1)/myPow(x,-n)
    }
    return recursion(x,n)
}

func recursion(x float64,n int) float64 {
    if n == 0 {
        return float64(1)
    }
    half := recursion(x,n / 2)
    if n % 2 == 0 {
        return half*half
    }else{
        return half*half*x
    }
}
```

#### [电话号码的字母组合](https://leetcode-cn.com/problems/letter-combinations-of-a-phone-number/)

```go
func letterCombinations(digits string) []string {
    res := make([]string,0)
    if len(digits) == 0 {
        return res
    }
    items := map[byte]string{
        '2':"abc",
        '3':"def",
        '4':"ghi",
        '5':"jkl",
        '6':"mno",
        '7':"pqrs",
        '8':"tuv",
        '9':"wxyz",
    }
    recursion(0,digits,"",&res,items)
    return res
}

func recursion(index int,digits string,s string,ret *[]string,items map[byte]string) {
    if index == len(digits) {
        *ret = append(*ret,s)
        return
    }
    characters := items[digits[index]]
    for _,v := range characters {
        recursion(index+1,digits,s+string(v),ret,items)
    }
}
```

#### [51. N 皇后](https://leetcode-cn.com/problems/n-queens/)

```go
import "strings"
var res [][]string

func isValid(board [][]string, row, col int) (res bool){
    n := len(board)
    for i:=0; i < row; i++ {
        if board[i][col] == "Q" {
            return false
        }
    }
    for i := 0; i < n; i++{
        if board[row][i] == "Q" {
            return false
        }
    }

    for i ,j := row, col; i >= 0 && j >=0 ; i, j = i - 1, j- 1{
        if board[i][j] == "Q"{
            return false
        }
    }
    for i, j := row, col; i >=0 && j < n; i,j = i-1, j+1 {
        if board[i][j] == "Q" {
            return false
        }
    }
    return true
}

func backtrack(board [][]string, row int) {
    size := len(board)
    if row == size{
        temp := make([]string, size)
        for i := 0; i<size;i++{
            temp[i] = strings.Join(board[i],"")
        }
        res =append(res,temp)
        return 
    }
    for col := 0; col < size; col++ {
        if !isValid(board, row, col){
            continue
        }
        board[row][col] = "Q"
        backtrack(board, row+1)
        board[row][col] = "."
    }
}

func solveNQueens(n int) [][]string {
    res = [][]string{}
    board := make([][]string, n)
    for i := 0; i < n; i++{
        board[i] = make([]string, n)
    }
    for i := 0; i < n; i++{
        for j := 0; j<n;j++{
            board[i][j] = "."
        }
    }
    backtrack(board, 0)

    return res
}
```

### 深度优先搜索与广度优先搜索

##### BFS

```
//不需要知道深度：
while queue 不空：
    cur = queue.pop()
    for 节点 in cur的所有相邻节点：
        if 该节点有效且未访问过：
            queue.push(该节点)

//需要知道深度：
level = 0
while queue 不空：
    size = queue.size()
    while (size --) {
        cur = queue.pop()
        for 节点 in cur的所有相邻节点：
            if 该节点有效且未被访问过：
                queue.push(该节点)
    }
    level ++;
```

##### DFS

```
def backtrack(待搜索的集合, 递归到第几层, 状态变量 1, 状态变量 2, 结果集):
    # 写递归函数都是这个套路：先写递归终止条件
    if 可能是层数够深了:
        # 打印或者把当前状态添加到结果集中
        return

    for 可以执行的分支路径 do           //分支路径
        
        # 剪枝
        if 递归到第几层, 状态变量 1, 状态变量 2, 符合一定的剪枝条件:
            continue

        对状态变量状态变量 1, 状态变量 2 的操作（#）
   
        # 递归执行下一层的逻辑
        backtrack(待搜索的集合, 递归到第几层, 状态变量 1, 状态变量 2, 结果集)

        对状态变量状态变量 1, 状态变量 2 的操作（与标注了 # 的那一行对称，称为状态重置）
        
    end for
```

#### [102. 二叉树的层序遍历](https://leetcode-cn.com/problems/binary-tree-level-order-traversal/)

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func levelOrder(root *TreeNode) [][]int {
    var res [][]int
    var stack []*TreeNode
    stack = append(stack, root)
    for len(stack) != 0 {
        n := len(stack)
        var v []int
        for i := 0;i < n;i ++ {  
            node := stack[0]
            stack = stack[1:]
            if node != nil {
                v = append(v, node.Val)
                if node.Left != nil {
                    stack = append(stack, node.Left)
                }
                if node.Right != nil {
                    stack = append(stack, node.Right)
                }
            }
        }
        if len(v) > 0 {
            res = append(res, v)
        }
    }
    return res
}
```

#### [433. 最小基因变化](https://leetcode-cn.com/problems/minimum-genetic-mutation/)

```
```

#### [200. 岛屿数量](https://leetcode-cn.com/problems/number-of-islands/)

```go
func numIslands(grid [][]byte) int {
    ret := 0
    m := len(grid)
    n := len(grid[0])
    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if grid[i][j] == '1' {
                ret++
                dfs(i, j, grid)
            }
        }
    }
    return ret
}

func dfs(x, y int, grid [][]byte) {
    if x >= 0 && x < len(grid) && y >= 0 && y < len(grid[0]) && grid[x][y] == '1' {
        grid[x][y] = '0'
        dfs(x+1, y, grid)
        dfs(x-1, y, grid)
        dfs(x, y+1, grid)
        dfs(x, y-1, grid)
    }
}
```

#### [733. 图像渲染](https://leetcode-cn.com/problems/flood-fill/)

```go
func floodFill(image [][]int, sr int, sc int, newColor int) [][]int {
    dx := []int{0,1,0,-1}
    dy := []int{1,0,-1,0}
    oldColor := image[sr][sc]
    if oldColor == newColor {
        return image
    }
    image[sr][sc] = newColor
    for i := 0; i < 4; i++ {
        x := sr + dx[i]
        y := sc + dy[i]
        if x >= 0 && x < len(image) && y >= 0 && y < len(image[0]) && image[x][y] == oldColor {
            floodFill(image,x,y,newColor)
        }
    }
    return image
}
```

#### [130. 被围绕的区域](https://leetcode-cn.com/problems/surrounded-regions/)

```go
func solve(board [][]byte)  {
    m := len(board)
    n := len(board[0])
    //上下两排
    for i := 0; i < n; i++ {
        if board[0][i] == 'O' {
            dfs(board,0,i)
        }
        if board[m-1][i] == 'O' {
            dfs(board,m-1,i)
        }
    }
    //左右两列
    for j := 0; j < m; j++ {
        if board[j][0] == 'O' {
            dfs(board,j,0)
        }
        if board[j][n-1] == 'O' {
            dfs(board,j,n-1)
        }
    }

    for a := 0; a < m; a++ {
        for b := 0; b < n; b++ {
            if board[a][b] == 'O' {
                board[a][b] = 'X'
            }
        }
    }

    for c := 0; c < m; c++ {
        for d := 0; d < n; d++ {
            if board[c][d] == 'Y' {
                board[c][d] = 'O'
            }
        }
    }
}

func dfs(board [][]byte,i,j int) {
    board[i][j] = 'Y'
    dx := []int{-1,0,1,0}
    dy := []int{0,1,0,-1}
    for z := 0; z < 4; z++ {
        x := i + dx[z]
        y := j + dy[z]
        if x >= 0 && x < len(board) && y >= 0 && y < len(board[0]) && board[x][y] == 'O' {
            dfs(board,x,y)
        }
    }
}
```



###贪心算法

#### [455. 分发饼干](https://leetcode-cn.com/problems/assign-cookies/)

```go
func findContentChildren(g []int, s []int) int {
  sort.Ints(g)
  sort.Ints(s)

  child := 0
  for sIdx := 0; child < len(g) && sIdx < len(s); sIdx++ {
    if s[sIdx] >= g[child] {
      child++
    }
  }

  return child
}
```

#### [122. 买卖股票的最佳时机 II](https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-ii/)

```go
func maxProfit(prices []int) int {
    res := 0
    for i := 1;i < len(prices);i++ {
        if prices[i] > prices[i-1] {
            res += (prices[i]-prices[i-1])
        }
    }
    return res
}
```

#### [55. 跳跃游戏](https://leetcode-cn.com/problems/jump-game/)

```go
func canJump(nums []int) bool {
    n := len(nums)
    if n == 0 {
        return false
    }
    canReachAble := n-1
    for i := n-1;i >= 0;i-- {
        if nums[i] + i >= canReachAble {
            canReachAble = i
        }
    }
    return canReachAble == 0
}
```

