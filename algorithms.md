# Алгоритмы — Вопросы и ответы

Раздел перенесён из основного README. Содержит вопросы по алгоритмам и реализации на языке Go.

---

<!-- Алгоритмы -->
<details>
<summary><b><i>Алгоритмы</i></b></summary>

---

- Реализации: [ Реализации алгоритмов на языке Go (смотреть при необходимости) ]

<details>
<summary>Реализации</summary>

---

  <!-- Сортировка -->
<details>
<summary>Сортировка</summary>

---

<details>
<summary>Quick Sort</summary>

 - Quick Sort  
   Сложность: O(n log n) в среднем и лучшем случае, O(n^2) в худшем случае.  
   In-place: Да.  
   Стабильность: Нет.

 - Когда использовать:  
   Для больших наборов данных.  
   Когда требуется быстрая "in-place" сортировка.  
   Когда стабильность сортировки не является ключевым фактором.

 ```go
 package main

 import "fmt"

 // quickSort сортирует подмассив arr[low:high] на месте.
 func quickSort(arr []int, low int, high int) {
     // Если индекс "low" больше или равен "high", прекратить выполнение.
     if low < high {
         // Находим индекс опорного элемента
         pivot := partition(arr, low, high)
         // Рекурсивно сортируем подмассивы слева и справа от опорного элемента
         quickSort(arr, low, pivot-1)
         quickSort(arr, pivot+1, high)
     }
}

 // partition выбирает опорный элемент и перераспределяет элементы так, чтобы
 // элементы меньше опорного находились слева, а больше — справа.
 func partition(arr []int, low int, high int) int {
     // Используем последний элемент в качестве опорного
     pivot := arr[high]
     // Инициализируем i как индекс, указывающий на самый левый элемент
     i := low - 1

     // Проходим через каждый элемент и сравниваем его с опорным
     for j := low; j < high; j++ {
         if arr[j] <= pivot {
             i++
             // Меняем местами arr[i] и arr[j]
             arr[i], arr[j] = arr[j], arr[i]
         }
     }

     // Помещаем опорный элемент на правильную позицию
     arr[i+1], arr[high] = arr[high], arr[i+1]
     return i + 1
 }

 func main() {
     arr := []int{9, 7, 5, 11, 12, 2, 14, 3, 10, 6}
     fmt.Println("Before sorting:", arr)
     quickSort(arr, 0, len(arr)-1)
     fmt.Println("After sorting:", arr)
 }
 ```  
</details>

<details>
<summary>Merge Sort</summary>

- Merge Sort  
  Сложность: O(n log n) во всех случаях.  
  In-place: Нет (требует дополнительную память).  
  Стабильность: Да.  

- Когда использовать:  
  Для больших наборов данных.  
  Когда нужна стабильная сортировка.  
  Когда нужна сортировка для связанных списков (без случайного доступа).
  Когда данные слишком велики для помещения в память (внешняя сортировка — Merge Sort хорошо подходит для сортировки блоков на диске).  

```go
package main

import "fmt"

// mergeSort рекурсивно разделяет и сортирует массив
func mergeSort(arr []int) []int {
    // Если массив состоит из одного или нуля элементов, он уже отсортирован
    if len(arr) <= 1 {
      return arr
    }

 // Разделяем массив на две половины
 middle := len(arr) / 2
 left := mergeSort(arr[:middle])
 right := mergeSort(arr[middle:])

 // Сливаем отсортированные половины
 return merge(left, right)
}

// merge сливает два отсортированных массива в один отсортированный массив
func merge(left, right []int) []int {
    // Инициализируем результат с предполагаемым размером для оптимизации
    result := make([]int, 0, len(left)+len(right))

 i, j := 0, 0
 // Проходим через каждый элемент в обоих массивах
 for i < len(left) && j < len(right) {
  if left[i] <= right[j] {
   result = append(result, left[i])
   i++
  } else {
   result = append(result, right[j])
   j++
  }
 }

 // Добавляем оставшиеся элементы, если таковые есть
 result = append(result, left[i:]...)
 result = append(result, right[j:]...)

 return result
}

func main() {
    arr := []int{9, 7, 5, 11, 12, 2, 14, 3, 10, 6}
    fmt.Println("Before sorting:", arr)
    arr = mergeSort(arr)
    fmt.Println("After sorting:", arr)
}
```
</details>

<details>
<summary>Bubble Sort</summary>

- Bubble Sort  
  Сложность: O(n^2) в среднем и худшем случае, O(n) в лучшем случае (когда массив уже отсортирован).  
  In-place: Да.  
  Стабильность: Да.  

- Когда использовать:  
  Для очень небольших массивов.  
  Когда почти отсортированный массив нуждается в небольшой корректировке.  
  Обучающие задачи или простые приложения.  

```go
package main

import "fmt"

// bubbleSort сортирует массив на месте, используя алгоритм пузырьковой сортировки.
func bubbleSort(arr []int) {
    n := len(arr)

 // Внешний цикл проходит по всему массиву
 for i := 0; i < n; i++ {
  swapped := false // Флаг для оптимизации

  // Внутренний цикл сравнивает каждую пару соседних элементов
  for j := 0; j < n-i-1; j++ {
   if arr[j] > arr[j+1] {
    // Меняем местами
    arr[j], arr[j+1] = arr[j+1], arr[j]
    swapped = true
   }
  }

  // Если не было обменов, массив уже отсортирован
  if !swapped {
   break
  }
 }
}

func main() {
    arr := []int{64, 34, 25, 12, 22, 11, 90}
    fmt.Println("Before sorting:", arr)
    bubbleSort(arr)
    fmt.Println("After sorting:", arr)
}
```
</details>

<details>
<summary>Insertion Sort</summary>

- Insertion Sort  
  Сложность: O(n^2) в среднем и худшем случае, O(n) в лучшем случае (когда массив уже отсортирован).  
  In-place: Да.  
  Стабильность: Да.

- Когда использовать:  
  Для небольших массивов.  
  Для почти отсортированных массивов.  
  В алгоритмах, которые требуют сортировки на ходу.  

```go
package main

import "fmt"

// insertionSort сортирует массив на месте, используя алгоритм сортировки вставками.
func insertionSort(arr []int) {
    n := len(arr)

 // Внешний цикл перебирает все элементы массива
 for i := 1; i < n; i++ {
  key := arr[i]  // Текущий элемент для вставки
  j := i - 1 // Индекс предыдущего элемента

  // Перемещаем все элементы, большие чем key, на одну позицию вперед
  for j >= 0 && arr[j] > key {
   arr[j+1] = arr[j]
   j--
  }

  // Вставляем key на правильное место
  arr[j+1] = key
 }
}

func main() {
    arr := []int{12, 11, 13, 5, 6}
    fmt.Println("Before sorting:", arr)
    insertionSort(arr)
    fmt.Println("After sorting:", arr)
}

```
</details>

<details>
<summary>Heap Sort</summary>

- Heap Sort  
  Сложность: O(n log n)  
  In-place: Да.  
  Стабильность: Нет. 

- Когда использовать:  
  Когда требуется быстрая сортировка без дополнительного использования памяти.  
  Когда стабильность не требуется.  

```go
package main

import "fmt"

// heapify - функция для преобразования поддерева в двоичную кучу
// с корнем в индексе i для данного массива arr длины n
func heapify(arr []int, n int, i int) {
    largest := i          // Инициализируем largest как корень
    l := 2*i + 1          // Левый = 2*i + 1
    r := 2*i + 2          // Правый = 2*i + 2

 // Если левый дочерний элемент больше корня
 if l < n && arr[l] > arr[largest] {
  largest = l
 }

 // Если правый дочерний элемент больше корня
 if r < n && arr[r] > arr[largest] {
  largest = r
 }

 // Если largest не корень
 if largest != i {
  arr[i], arr[largest] = arr[largest], arr[i]

  // Рекурсивно преобразуем в кучу поддерево с корнем в largest
  heapify(arr, n, largest)
 }
}

// heapSort - функция для сортировки массива длины n
func heapSort(arr []int, n int) {
    // Построение кучи (перегруппировка массива)
    for i := n / 2 - 1; i >= 0; i-- {
    heapify(arr, n, i)
    }

 // Один за другим извлекаем элементы из кучи
 for i := n - 1; i >= 0; i-- {
  // Перемещаем текущий корень в конец
  arr[0], arr[i] = arr[i], arr[0]

  // Вызываем heapify на уменьшенной куче
  heapify(arr, i, 0)
 }
}

func main() {
    arr := []int{12, 11, 13, 5, 6, 7}
    n := len(arr)
 fmt.Println("Before sorting:", arr)
 heapSort(arr, n)
 fmt.Println("After sorting:", arr)
}
```
</details>

---

</details>

  <!-- Поиск -->
<details>
<summary>Поиск</summary>

---

<details>
<summary>Binary Search</summary>

- Binary Search (Бинарный поиск)  
  Сложность: O(log n) в среднем и худшем случае, где n - размер массива. O(1) — в лучшем случае (когда элемент найден в центре массива).
  In-place: Да.
  Стабильность: Не применимо.

- Когда использовать:
  Когда массив упорядочен и требуется найти элемент.
  При работе с большими отсортированными данными, где бинарный поиск эффективнее линейного поиска.

```go
package main

import "fmt"

// binarySearch выполняет бинарный поиск элемента в упорядоченном массиве.
func binarySearch(arr []int, target int) int {
    low, high := 0, len(arr)-1

 for low <= high {
  mid := low + (high-low)/2 // Находим средний индекс

  if arr[mid] == target {
   return mid // Найден элемент, возвращаем его индекс
  } else if arr[mid] < target {
   low = mid + 1 // Искомый элемент находится в правой половине
  } else {
   high = mid - 1 // Искомый элемент находится в левой половине
  }
 }

 return -1 // Элемент не найден
}

func main() {
    arr := []int{2, 4, 6, 8, 10, 12, 14, 16}
    target := 10
    index := binarySearch(arr, target)

 if index != -1 {
  fmt.Printf("%d found at index %d\n", target, index)
 } else {
  fmt.Printf("%d not found in the array\n", target)
 }
}

```
</details>

<details>
<summary>Linear Search</summary>

- Linear Search (Линейный поиск)
  Сложность: O(n) в среднем и худшем случае, O(1) в лучшем случае (когда элемент найден первым). Где n - размер массива.
  In-place: Да.
  Стабильность: Не применимо

- Когда использовать:
  Когда массив небольшой или не упорядочен.
  Для поиска элемента в небольших наборах данных.
  Когда другие алгоритмы не применимы или неэффективны из-за малого размера данных.

```go
package main

import "fmt"

// linearSearch выполняет линейный поиск элемента в массиве.
func linearSearch(arr []int, target int) int {
    for i, num := range arr {
        if num == target {
            return i // Найден элемент, возвращаем его индекс
        }
    }
    return -1 // Элемент не найден
}

func main() {
    arr := []int{2, 4, 6, 8, 10, 12, 14, 16}
    target := 10
    index := linearSearch(arr, target)

 if index != -1 {
  fmt.Printf("%d found at index %d\n", target, index)
 } else {
  fmt.Printf("%d not found in the array\n", target)
 }
}

```
</details>

<details>
<summary>Depth-First Search (DFS)</summary>

- Depth-First Search (DFS поиск в глубину)
  Сложность: O(V + E), где V - количество вершин, E - количество рёбер.
  In-place: Нет.
  Стабильность: Не применимо.

- Когда использовать:
  Для обхода графов в глубину и поиска путей в графах.
  Когда нужно исследовать все возможные варианты обхода в глубину, например, в алгоритмах поиска в глубину или поиска путей.

```go
package main

import (
"fmt"
)

// Graph: представляет собой граф в виде карты, где ключ - это вершина, а значение - это список смежных вершин.
type Graph map[int][]int

// DFS: обходит граф в глубину начиная с вершины start.
func DFS(graph Graph, start int, visited map[int]bool) {
    // Отмечаем текущую вершину как посещенную
    visited[start] = true
    fmt.Println(start)

 // Перебираем всех соседей текущей вершины
 for _, neighbor := range graph[start] {
  if !visited[neighbor] { // Если сосед не был посещен
   DFS(graph, neighbor, visited) // Рекурсивно вызываем DFS для соседа
  }
 }
}

func main() {
    graph := Graph{
    0: {1, 2},
    1: {2},
    2: {0, 3},
    3: {3},
    }

 visited := make(map[int]bool)

 fmt.Println("Depth-First Search:")
 DFS(graph, 2, visited)
}

```
</details>

<details>
<summary>Breadth-First Search (BFS)</summary>

- Breadth-First Search (BFS поиск в ширину)
  Сложность: O(V + E), где V - количество вершин, E - количество рёбер.
  In-place: Нет.
  Стабильность: Не применимо.

- Когда использовать:
  Для поиска кратчайшего пути в графе.
  Когда нужно исследовать все соседние вершины на текущем уровне графа (например, в задачах поиска кратчайшего пути).
  Когда нужен полный обход графа в ширину.

```go
package main

import (
    "fmt"
    "container/list"
)

// Graph представляет собой граф в виде карты, где ключ - это вершина, а значение - это список смежных вершин.
type Graph map[int][]int

// BFS обходит граф в ширину начиная с вершины start.
func BFS(graph Graph, start int) {
    // Создаем очередь и добавляем в нее стартовую вершину
    queue := list.New()
    queue.PushBack(start)

 // Инициализируем множество посещенных вершин
 visited := make(map[int]bool)
 visited[start] = true

 fmt.Println("Breadth-First Search:")

 for queue.Len() > 0 {
  // Извлекаем вершину из начала очереди
  element := queue.Front()
  vertex := element.Value.(int)
  queue.Remove(element)

  fmt.Println(vertex)

  // Добавляем все непосещенные соседние вершины в очередь
  for _, neighbor := range graph[vertex] {
   if !visited[neighbor] {
    queue.PushBack(neighbor)
    visited[neighbor] = true
   }
  }
 }
}

func main() {
    graph := Graph{
    0: {1, 2},
    1: {2},
    2: {0, 3},
    3: {3},
    }

 fmt.Println("Breadth-First Search:")
 BFS(graph, 2)
}

```
</details>

---

</details>

  <!-- Динамическое программирование -->
<details>
<summary>Динамическое программирование</summary>

---

<details>
<summary>Fibonacci Series</summary>

- Сложность: O(n) в среднем, лучшем и худшем случае.
  In-place: Нет (требует дополнительное пространство O(n)).
  Стабильность: Не применимо.

- Когда использовать: Когда требуется вычислить n-ое число Фибоначчи.

```go
package main

import "fmt"

// fibonacci использует динамическое программирование для вычисления n-го числа Фибоначчи
func fibonacci(n int) int {
    // Если n равно 0 или 1, возвращаем само число
    if n <= 1 {
        return n
    }
    // Инициализация слайса для сохранения чисел Фибоначчи
    fib := make([]int, n+1)
    fib[1] = 1
    // Вычисляем числа Фибоначчи начиная с 2 до n
    for i := 2; i <= n; i++ {
        fib[i] = fib[i-1] + fib[i-2]
    }
    return fib[n]
}

func main() {
    n := 10 // Можно изменить на любое желаемое число
    fmt.Println(fibonacci(n))
}
```
</details>

<details>
<summary>Knapsack Problem</summary>

- Сложность: O(nW) в среднем, лучшем и худшем случае, где n - количество предметов, а W - вместимость рюкзака.
  In-place: Нет.
  Стабильность: Не применимо.

- Когда использовать: Когда требуется определить максимальное значение, которое можно получить, размещая предметы с ограниченной вместимостью в рюкзаке.

```go
package main

import "fmt"

// Вспомогательная функция для получения максимального значения
func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

// 0-1 knapsack problem решение задачи о рюкзаке
func knapsack(values, weights []int, capacity int) int {
    n := len(values)
    // Инициализация двумерного слайса для динамического программирования
    dp := make([][]int, n+1)
    for i := range dp {
        dp[i] = make([]int, capacity+1)
    }

    // Заполнение таблицы dp
    for i := 1; i <= n; i++ {
        for w := 1; w <= capacity; w++ {
            if weights[i-1] <= w {
                dp[i][w] = max(dp[i-1][w], values[i-1]+dp[i-1][w-weights[i-1]])
            } else {
                dp[i][w] = dp[i-1][w]
            }
        }
    }
    return dp[n][capacity]
}

func main() {
    values := []int{60, 100, 120}  // Значения предметов
    weights := []int{10, 20, 30}  // Веса предметов
    capacity := 50                // Вместимость рюкзака
    fmt.Println(knapsack(values, weights, capacity))
}
```
</details>

<details>
<summary>Longest Common Subsequence</summary>

- Сложность: O(mn) в среднем, лучшем и худшем случае, где m и n - длины двух строк.
  In-place: Нет.
  Стабильность: Не применимо.

- Когда использовать: Когда нужно определить наибольшую общую подпоследовательность между двумя строками.

```go
package main

import "fmt"

// Вспомогательная функция для получения максимального значения
func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

// lcs находит наибольшую общую подпоследовательность двух строк
func lcs(X, Y string) int {
    m := len(X)
    n := len(Y)
    // Инициализация двумерного слайса для динамического программирования
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }

    // Заполнение таблицы dp
    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if X[i-1] == Y[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}

func main() {
    str1 := "AGGTAB"
    str2 := "GXTXAYB"
    fmt.Println(lcs(str1, str2))
}
```
</details>

<details>
<summary>Coin Change Problem</summary>

- Сложность: O(amount * n) в среднем, лучшем и худшем случае, где n - количество различных монет.
  In-place: Нет (требует дополнительное пространство O(amount)).
  Стабильность: Не применимо.

- Когда использовать: Когда нужно определить минимальное количество монет, которые необходимо использовать, чтобы собрать заданную сумму.

```go
package main

import (
    "fmt"
    "math"
)

// coinChange решает задачу о размене монет
func coinChange(coins []int, amount int) int {
    // Инициализация слайса для динамического программирования
    dp := make([]int, amount+1)
    for i := range dp {
        dp[i] = math.MaxInt32
    }
    dp[0] = 0

    // Заполнение таблицы dp
    for _, coin := range coins {
        for i := coin; i <= amount; i++ {
            dp[i] = min(dp[i], dp[i-coin]+1)
        }
    }

    if dp[amount] == math.MaxInt32 {
        return -1
    }
    return dp[amount]
}

// Вспомогательная функция для получения минимального значения
func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func main() {
    coins := []int{1, 2, 5}
    amount := 11
    fmt.Println(coinChange(coins, amount))
}
```
</details>

---

</details>

  <!-- Графы -->
<details>
<summary>Графы</summary>

---

<details>
<summary>Dijkstra's Algorithm</summary>

- Сложность:  
  В среднем: O(V*V), но с приоритетной очередью или кучей: O((V + E) log V).
  Лучший случай: O((V + E) log V) с кучей  
  In-place: Да.  
  Стабильность: Не применимо.  
- Когда использовать: Когда требуется найти кратчайшие пути от одной начальной вершины ко всем другим вершинам в взвешенном графе.

```go
package main

import (
    "fmt"
    "math"
)

// Определение константы для представления бесконечности
const Infinity = math.MaxInt64

// Функция для вычисления кратчайших путей от начальной вершины до всех остальных
func dijkstra(graph [][]int, start int) []int {
    n := len(graph)
    visited := make([]bool, n) // массив для отметки посещенных вершин
    dist := make([]int, n)     // массив для хранения расстояния от стартовой вершины до каждой другой
    for i := 0; i < n; i++ {
        dist[i] = Infinity // инициализация расстояний как бесконечности
    }
    dist[start] = 0 // расстояние до стартовой вершины равно 0

 for i := 0; i < n-1; i++ {
  u := minDistance(dist, visited) // выбор вершины с минимальным расстоянием из непосещенных
  visited[u] = true                // помечаем эту вершину как посещенную

  // Обновление значений расстояния для соседних вершин выбранной вершины
  for v := 0; v < n; v++ {
   // Проходим только по непосещенным вершинам с ненулевым весом ребра и обновляем расстояние, если найден более короткий путь
   if !visited[v] && graph[u][v] != 0 && dist[u] != Infinity && dist[u]+graph[u][v] < dist[v] {
    dist[v] = dist[u] + graph[u][v]
   }
  }
 }
 return dist
}

// Вспомогательная функция для нахождения вершины с минимальным расстоянием из непосещенных
func minDistance(dist []int, visited []bool) int {
    minim := Infinity // начальное значение минимального расстояния
    minIndex := -1  // индекс вершины с минимальным расстоянием

 for i := 0; i < len(dist); i++ {
  if dist[i] < minim && !visited[i] {
   minim = dist[i]
   minIndex = i
  }
 }
 return minIndex
}

func main() {
    graph := [][]int{
    // представление графа в виде матрицы смежности
    {0, 4, 0, 0, 0, 0, 0, 8, 0},
    // ... (и так далее для каждой строки матрицы)
    }
    // Вывод результатов работы алгоритма Дейкстры для стартовой вершины 0
    fmt.Println(dijkstra(graph, 0))
}

```
</details>

<details>
<summary>Floyd-Warshall Algorithm</summary>

- Сложность:  
  В среднем: O(V * V * V)  
  Лучший случай: O(V * V * V)  
  Худший случай: O(V * V * V)  
  In-place: Да.  
  Стабильность: Не применимо.  
- Когда использовать: Когда нужно найти кратчайшие пути между всеми парами вершин в графе.

```go
package main

import (
    "fmt"
    "math"
)

// Инициализация констант для представления бесконечности
const Infinity = math.MaxInt64
const NULL = -1

// Функция реализующая алгоритм Флойда-Уоршалла для нахождения кратчайших путей между всеми парами вершин
func floydWarshall(graph [][]int) [][]int {
    n := len(graph)
    dist := make([][]int, n) // матрица для хранения кратчайших путей между вершинами
    for i := 0; i < n; i++ {
        dist[i] = make([]int, n)
        for j := 0; j < n; j++ {
            if i == j {
                dist[i][j] = 0 // расстояние от вершины до самой себя равно 0
            } else if graph[i][j] != 0 {
                dist[i][j] = graph[i][j] // если есть прямое ребро, используем его вес
            } else {
                dist[i][j] = Infinity // иначе устанавливаем бесконечность
            }
        }
}

 // Проход по всем вершинам и обновление кратчайших путей
 for k := 0; k < n; k++ {
  for i := 0; i < n; i++ {
   for j := 0; j < n; j++ {
    // Если путь через вершину k короче, чем текущий путь, обновляем значение
    if dist[i][k]+dist[k][j] < dist[i][j] {
     dist[i][j] = dist[i][k] + dist[k][j]
    }
   }
  }
 }
 return dist
}

func main() {
    graph := [][]int{
    // представление графа в виде матрицы смежности
    {0, 5, Infinity, 10},
    // ... (и так далее для каждой строки матрицы)
    }
    // Вывод результатов работы алгоритма Флойда-Уоршалла
    result := floydWarshall(graph)
    for _, row := range result {
        fmt.Println(row)
    }
}

```
</details>

<details>
<summary>Kruskal's Algorithm</summary>

- Сложность:  
  В среднем: O(E logE) или O(E logV)  
  Лучший случай: O(E logE) или O(E logV)  
  Худший случай: O(E logE) или O(E logV)  
  In-place: Нет.  
  Стабильность: Не применимо.  
- Когда использовать: Для нахождения минимального остовного дерева во взвешенном графе.

```go
package main

import (
    "fmt"
    "sort"
)

type Edge struct {
src, dest, weight int
}

// Для создания и управления множествами
type Subset struct {
parent, rank int
}

// Находит представителя множества элемента i
func find(subsets []Subset, i int) int {
    if subsets[i].parent != i {
        subsets[i].parent = find(subsets, subsets[i].parent)
    }
    return subsets[i].parent
}

// Объединяет два множества по рангу
func union(subsets []Subset, x, y int) {
    xroot := find(subsets, x)
    yroot := find(subsets, y)

 // Прикрепляем дерево меньшего ранга к корню дерева с большим рангом
 if subsets[xroot].rank < subsets[yroot].rank {
  subsets[xroot].parent = yroot
 } else if subsets[xroot].rank > subsets[yroot].rank {
  subsets[yroot].parent = xroot
 } else {
  subsets[yroot].parent = xroot
  subsets[xroot].rank++
 }
}

// Алгоритм Краскала для поиска минимального остовного дерева
func kruskal(edges []Edge, vertices int) []Edge {
    sort.Slice(edges, func(i, j int) bool {
    return edges[i].weight < edges[j].weight
    }) // Сортируем ребра по весу

 subsets := make([]Subset, vertices)
 for i := 0; i < vertices; i++ {
  subsets[i] = Subset{parent: i, rank: 0}
 }

 var mst []Edge
 e := 0 // Индекс для mst[]

 for _, edge := range edges {
  x := find(subsets, edge.src)
  y := find(subsets, edge.dest)

  if x != y { // Если это не формирует цикл
   mst = append(mst, edge)
   union(subsets, x, y)
   e++
  }
  if e == vertices-1 {
   break
  }
 }
 return mst
}
```
</details>

<details>
<summary>Prim's Algorithm</summary>

  - Сложность:  
    В среднем: O(V * V)  
    Лучший случай: O((E+V) log V) c кучей  
    Худший случай: O(V * V)
    In-place: Нет.  
    Стабильность: Не применимо.  
  - Когда использовать: Для нахождения минимального остовного дерева во взвешенном графе.

  ```go
  package main

  import (
   "fmt"
   "math"
  )

  // Определение константы для представления бесконечности
  const Infinity = math.MaxInt64

  // Функция для нахождения вершины с минимальным ключом из множества вершин, не включенных в MST
  func minKey(key []int, mstSet []bool) int {
   minim := Infinity
   minIndex := 0

   for v := 0; v < len(key); v++ {
    if mstSet[v] == false && key[v] < minim {
     minim = key[v]
     minIndex = v
    }
   }
   return minIndex
  }

  // Функция для вывода ребер и их веса минимального остовного дерева
  func printMST(parent []int, graph [][]int) {
   for i := 1; i < len(graph); i++ { 
          fmt.Printf("Ребро %d - %d \t Вес: %d \n", parent[i], i, graph[i][parent[i]])
      }
  }

  // Функция для построения и вывода MST для графа, представленного в виде матрицы смежности
  func primMST(graph [][]int) {
      n := len(graph)
      key := make([]int, n)   // Значения, используемые для выбора минимального ребра для каждой вершины
      parent := make([]int, n) // Array to store constructed MST
      mstSet := make([]bool, n)

      // Инициализация всех ключей как бесконечность
      for i := 0; i < n; i++ {
          key[i] = Infinity
      }

      key[0] = 0     // Делаем ключ 0 так, чтобы эта вершина была выбрана первой
      parent[0] = -1 // Первый узел всегда корень нашего MST

      for count := 0; count < n-1; count++ {
          // Выбираем минимальный ключ из множества вершин, которые еще не включены в MST
          u := minKey(key, mstSet)
          mstSet[u] = true // Добавляем выбранную вершину в mstSet

          // Обновляем значения ключа и индекса родителя для вершин, смежных с выбранной вершиной
          for v := 0; v < n; v++ {
              if graph[u][v] != 0 && mstSet[v] == false && graph[u][v] < key[v] {
                  parent[v] = u
                  key[v] = graph[u][v]
              }
          }
}

      printMST(parent, graph)
  }

  func main() {
      graph := [][]int{
          {0, 2, 0, 6, 0},
          {2, 0, 3, 8, 5},
          {0, 3, 0, 0, 7},
          {6, 8, 0, 0, 9},
          {0, 5, 7, 9, 0},
      }
      primMST(graph)
  }
  ```
</details>

---

</details>

  <!-- Строки -->
<details>
<summary>Строки</summary>

---

<details>
<summary>KMP Algorithm</summary>

- Сложность:  
  В среднем: O(n+m)  
  Лучший случай: O(n) (если подстрока отсутствует)  
  Худший случай: O(n+m)  
  In-place: Да  
  Стабильность: Не применимо (это алгоритм поиска)  
- Когда использовать: Когда нужен быстрый поиск подстроки в строке.

```go
package main

import (
    "fmt"
)

// Функция для вычисления префикс-функции для KMP
func computeLPSArray(pattern string) []int {
    length := 0
    lps := make([]int, len(pattern))

    i := 1
    // Проход по всей строке для вычисления lps[i]
    for i < len(pattern) {
        if pattern[i] == pattern[length] {
            length++
            lps[i] = length
            i++
        } else {
            if length != 0 {
                // Найдено несовпадение, следует уменьшить length
                length = lps[length-1]
            } else {
                lps[i] = 0
                i++
            }
        }
    }
    return lps
}

// Функция KMP поиска
func KMPSearch(pat, txt string) {
    m := len(pat)
    n := len(txt)
    lps := computeLPSArray(pat)

    i, j := 0, 0
    // Проход по главной строке
    for i < n {
     if pat[j] == txt[i] {
         i++
         j++
     }
     // Если найдено полное совпадение
     if j == m {
         fmt.Printf("Найден шаблон на индексе %d\n", i-j)
         j = lps[j-1]
     } else if i < n && pat[j] != txt[i] {
         if j != 0 {
             j = lps[j-1]
         } else {
             i++
         }
     }
 }
}

func main() {
    txt := "ABABDABACDABABCABAB"
    pat := "ABABCABAB"
    KMPSearch(pat, txt)
}
```
</details>

<details>
<summary>Rabin-Karp Algorithm</summary>

- Сложность:  
  В среднем: O(n+m)  
  Лучший случай: O(n+m)  
  Худший случай: O(nm) при плохом хеше  
  In-place: Да  
  Стабильность: Не применимо (это алгоритм поиска)  
- Когда использовать: Когда нужен алгоритм на основе хеширования для поиска подстроки.

```go
package main

import (
    "fmt"
)

const base = 256 // основание для хеширования
const prime = 101 // простое число для модуляции хеша

// Функция поиска с использованием Rabin-Karp
func rabinKarpSearch(pat, txt string) {
    m := len(pat)
    n := len(txt)
    i, j := 0, 0
    pHash, tHash := 0, 0
    h := 1

    // Вычисление h^(m-1) для последующего использования
    for i = 0; i < m-1; i++ {
        h = (h * base) % prime
    }

    // Вычисление хешей для pat и первого окна txt
    for i = 0; i < m; i++ {
        pHash = (base*pHash + int(pat[i])) % prime
        tHash = (base*tHash + int(txt[i])) % prime
    }

    // Скользящее окно по тексту txt
    for i = 0; i <= n-m; i++ {
        // Сравнение хешей текущего окна txt и pat
        if pHash == tHash {
            // При совпадении хешей сравниваем символы
            for j = 0; j < m; j++ {
                if txt[i+j] != pat[j] {
                    break
                }
            }
            // Если pat[0...m-1] = txt[i, i+1, ...i+m-1]
            if j == m {
                fmt.Printf("Найден шаблон на индексе %d\n", i)
            }
        }

        // Вычисление хеша для следующего окна txt
        if i < n-m {
            tHash = (base*(tHash-int(txt[i])*h) + int(txt[i+m])) % prime
            if tHash < 0 {
                tHash += prime
            }
        }
    }
}

func main() {
    txt := "ABABDABACDABABCABAB"
    pat := "ABABCABAB"
    rabinKarpSearch(pat, txt)
}
```
</details>

<details>
<summary>Z-Algorithm</summary>

- Сложность:  
  В среднем: O(n+m)  
  Лучший случай: O(n+m)  
  Худший случай: O(n+m)  
  In-place: Да  
  Стабильность: Не применимо (это алгоритм поиска)  
- Когда использовать: Когда нужен алгоритм для поиска подстроки, основанный на Z-массиве.

```go
package main

import (
    "fmt"
)

// Функция для вычисления Z-массива
func getZArray(str string) []int {
    n := len(str)
    z := make([]int, n)

    l, r, k := 0, 0, 0
    for i := 1; i < n; i++ {
        // Если i находится за пределами текущего Z-окна, вычисляем Z[i] "с нуля"
        if i > r {
            l, r = i, i
            for r < n && str[r-l] == str[r] {
                r++
            }
            z[i] = r - l
            r--
        } else {
            // Иначе применяем оптимизацию к вычислению Z[i]
            k = i - l
            if z[k] < r-i+1 {
                z[i] = z[k]
            } else {
                l = i
                for r < n && str[r-l] == str[r] {
                    r++
                }
                z[i] = r - l
                r--
            }
        }
    }
    return z
}

// Функция для поиска с использованием Z-алгоритма
func zSearch(text, pattern string) {
    // Конкатенация текста и паттерна
    concat := pattern + "$" + text
    z := getZArray(concat)
    // Обход полученного Z-массива
    for i := 0; i < len(z); i++ {
        // Если значение z[i] равно длине паттерна, значит, паттерн найден на этой позиции
        if z[i] == len(pattern) {
            fmt.Printf("Найден шаблон на индексе %d\n", i-len(pattern)-1)
        }
    }
}

func main() {
    txt := "ABABDABACDABABCABAB"
    pat := "ABABCABAB"
    zSearch(txt, pat)
}
```
</details>

---

</details>

  <!-- Хэширование -->
<details>
<summary>Хэширование</summary>

---

<details>
<summary>Hashing Algorithms</summary>

- Реализации алгоритмов хеширования очень объемны. По этому рекомендую ознакомиться с ними самостоятельно

</details>

---

</details>

</details>

---

### Вопрос №1: [ Какие часто используемые алгоритмы ты знаешь? ]

<details>
<summary>Ответ</summary>

* Сортировка
  - Quick Sort — быстрая сортировка
  - Merge Sort — сортировка слиянием
  - Bubble Sort — сортировка пузырьком
  - Insertion Sort — сортировка вставками
  - Heap Sort — сортировка кучей
* Поиск
  - Binary Search — двоичный поиск
  - Linear Search — линейный поиск
  - Depth-First Search (DFS) — поиск в глубину
  - Breadth-First Search (BFS) — поиск в ширину
* Динамическое программирование
  - Fibonacci Series — вычисление чисел Фибоначчи
  - Knapsack Problem — задача о рюкзаке
  - Longest Common Subsequence — наибольшая общая подпоследовательность
  - Coin Change Problem — задача о размене монет
* Графы
  - Dijkstra's Algorithm — алгоритм Дейкстры для нахождения кратчайшего пути
  - Floyd-Warshall Algorithm — алгоритм Флойда—Уоршелла
  - Kruskal's Algorithm — алгоритм Краскала для минимального остовного дерева
  - Prim's Algorithm — алгоритм Прима для минимального остовного дерева
* Строки
  - KMP Algorithm — алгоритм Кнута—Морриса—Пратта для поиска подстроки
  - Rabin-Karp Algorithm — алгоритм Рабина—Карпа для поиска подстроки
  - Z-Algorithm — для поиска подстроки
* Хеширование
  - MD5 (Message Digest Algorithm 5):
    Производит 128-битный хеш.  
    Широко использовался для проверки целостности данных.  
    Сейчас считается устаревшим из-за потенциальных уязвимостей.  
  - SHA (Secure Hash Algorithms):
    - SHA-0: Первоначальная версия, быстро была заменена из-за наличия уязвимостей.
    - SHA-1: Производит 160-битный хеш. В настоящее время считается устаревшим из-за возможности коллизий.
    - SHA-2: Семейство функций, включая SHA-224, SHA-256, SHA-384, SHA-512, SHA-512/224, SHA-512/256. Это более безопасные варианты по сравнению с SHA-1.
    - SHA-3: Последний и наиболее современный член семейства Secure Hash Algorithms. Отличается от SHA-2.
  - CRC32 (Cyclic Redundancy Check):  
    Используется в сетевых коммуникациях для обнаружения изменений в сырых данных.
  - bcrypt:  
    Основан на Blowfish-шифровании.  
    Широко используется для хеширования паролей из-за его возможности "соления" и адаптивной природы.
  - scrypt:  
    Используется для хеширования паролей.  
    Основан на потреблении памяти, что делает его устойчивым к атакам на основе специализированного оборудования.
  - argon2:  
    Победитель Password Hashing Competition в 2015 году.  
    Имеет несколько вариантов: argon2i (устойчив к side-channel атакам) и argon2d (устойчив к атакам time-memory tradeoff).
  - Murmur и CityHash:  
    Не предназначены для криптографического использования.  
    Быстро генерируют хеш для использования в таких структурах данных, как хеш-таблицы.
  - BLAKE2:  
    Криптографическая хеш-функция, которая быстрее, чем MD5, SHA-1 и SHA-2.  
  - Whirlpool:  
    Криптографическая функция, производящая 512-битный хеш.  
  - RIPEMD (RACE Integrity Primitives Evaluation Message Digest):

</details>

---

### Вопрос №2: [ Как отсортировать файл на 100GB с 1GB ОЗУ? ]

<details>
<summary>Ответ</summary>

* Используйте внешнюю сортировку:
* Разделите большой файл на меньшие части размером < 1GB.
* Отсортируйте каждую часть в памяти и сохраните на диск.
* Объедините отсортированные части, считывая и сравнивая первые элементы каждого файла.

</details>

---

</details>
