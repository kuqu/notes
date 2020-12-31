
# 常见排序算法

{docsify-updated}

```
def bubble(arr):
	#冒泡排序，最易考
	if arr is None:
		return None
	for i in range(0, len(arr) - 1):
		for j in range(0, len(arr)  - 1 - i):
			if arr[j] > arr[j + 1]:
				arr[j], arr[j + 1] = arr[j + 1], arr[j]
	return arr


def selection(arr):
	# 选择排序，最直观的排序
	if arr is None:
		return None
	for i in range(0, len(arr) - 1):
		min = i
		for j in range(min + 1, len(arr)):
			if arr[min] > arr[j]:
				min = j
		arr[i], arr[min] = arr[min], arr[i]
	return arr


def insertion(arr):
	#插入排序
	if arr is None:
		return None
	if len(arr) == 1:
		return arr
	for i in range(1, len(arr)):
		while i-1 >= 0 and arr[i-1] > arr[i]:
			arr[i-1], arr[i] = arr[i], arr[i-1]
			i = i - 1
	return arr


def shell(arr):
	# 希尔排序，第一次突破n方
	if arr is None:
		return None
	gap = 1
	if gap < len(arr) // 3:
		gap = len(arr) // 3
	while gap >= 1:
		for i in range(0, gap): 
			for j in range(gap, len(arr), gap):
				while j - gap >= 0 and arr[j] < arr[j - gap]:
					arr[j], arr[j - gap] = arr[j - gap], arr[j]
					j -= gap
		gap -= 1
	return arr


def mergeSort(arr):
	# 归并排序，运用递归和分治好理解
	if arr is None:
		return None
	if len(arr) <= 1:
		return arr
	def merge(left, right):
		ret = []
		i = j = 0
		while i < len(left) or j < len(right):
			if j == len(right):
				ret.append(left[i])
				i += 1
			elif i == len(left):
				ret.append(right[j])
				j += 1
			else:
				if left[i] < right[j]:
					ret.append(left[i])
					i += 1
				else:
					ret.append(right[j])
					j += 1
		return ret
	return merge(mergeSort(arr[:len(arr)//2]), mergeSort(arr[len(arr)//2:]))


def quick(arr):
	# 快速排序
	if arr is None:
		return None
	if len(arr) <= 1:
		return arr
	middle = arr[0]
	left = []
	right = []
	for i in range(1, len(arr)):
		if middle > arr[i]:
			left.append(arr[i])
		else:
			right.append(arr[i])
	return quick(left) + [middle] + quick(right)


if __name__ == '__main__':

	arr =  [1, 4, 2, 6, 8, 10, 29, -55, 33, 100, 0, 3, 6]

	print(bubble(arr))

	print(selection(arr))

	print(insertion(arr))

	print(shell(arr))

	print(mergeSort(arr))

	print(quick(arr))
```


