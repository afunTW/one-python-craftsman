# Python 工匠：容器的門道

## 序言

> 這是 “Python 工匠”系列的第 4 篇文章。[[檢視系列所有文章]](https://github.com/piglei/one-python-craftsman)

<div style="text-align: center; color: #999; margin: 14px 0 14px;font-size: 12px;">
<img src="https://www.zlovezl.cn/static//uploaded/2019/01/6002476959_cca2bf5424_b_thumb.jpg" width="100%" /><div>
圖片來源: <a href="https://www.flickr.com/photos/chiotsrun/6002476959/in/photolist-a9qgh4-W4eQ1j-7MrCfo-4ARLWp-dwCzHh-Tascu9-RNRbRf-foLHW5-22dkkHM-9ceFA8-aGGd3a-26X3sqQ-iuTwX9-q52ktA-osn2eb-29oujY-6mXd1c-8E92nc-mPbq55-9GuPU8-26Q1NZG-8UL8PL-pdyFsW-7V8ifD-VZavJ8-2cUdHbU-9WrgjZ-6g7M5K-VMLVrb-cXDd4-bygFJG-C76kP-nMQW54-7MoQqn-qA3fud-c92dBU-tAzTBm-7KqFXc-24VvcW1-djQX9e-5LzjkA-63U4kb-bt1EEY-jLRpKo-dQSWBH-aDbqXc-8KhfnE-2m5ZsF-6ciuiR-qwdbt">"The Humble Mason Jar" by Chiot's Run</a> - 非商業性使用 2.0 通用</div>
</div>

容器”這兩個字很少被 Python 技術文章提起。一看到“容器”，大家想到的多是那頭藍色小鯨魚：*Docker*，但這篇文章和它沒有任何關係。本文裡的容器，是 Python 中的一個抽象概念，是對**專門用來裝其他物件的資料型別**的統稱。

在 Python 中，有四類最常見的內建容器型別：`列表（list）`、`元組（tuple）`、`字典（dict）`、`集合（set）`。透過單獨或是組合使用它們，可以高效的完成很多事情。

Python 語言自身的內部實現細節也與這些容器型別息息相關。比如 Python 的類例項屬性、全域性變數 `globals()` 等就都是透過字典型別來儲存的。

在這篇文章裡，我首先會從容器型別的定義出發，嘗試總結出一些日常編碼的最佳實踐。之後再圍繞各個容器型別提供的特殊機能，分享一些程式設計的小技巧。

### 內容目錄

* [底層看容器](#底層看容器)
    * [寫更快的程式碼](#寫更快的程式碼)
        * [1. 避免頻繁擴充列表/建立新列表](#1-避免頻繁擴充列表建立新列表)
        * [2. 在列表頭部操作多的場景使用 deque 模組](#2-在列表頭部操作多的場景使用-deque-模組)
        * [3. 使用集合/字典來判斷成員是否存在](#3-使用集合字典來判斷成員是否存在)
* [高層看容器](#高層看容器)
    * [寫擴充套件性更好的程式碼](#寫擴充套件性更好的程式碼)
        * [面向容器介面程式設計](#面向容器介面程式設計)
* [常用技巧](#常用技巧)
    * [1. 使用元組改善分支程式碼](#1-使用元組改善分支程式碼)
    * [2. 在更多地方使用動態解包](#2-在更多地方使用動態解包)
    * [3. 使用 next() 函式](#3-使用-next-函式)
    * [4. 使用有序字典來去重](#4-使用有序字典來去重)
* [常見誤區](#常見誤區)
    * [1. 當心那些已經枯竭的迭代器](#1-當心那些已經枯竭的迭代器)
    * [2. 別在迴圈體內修改被迭代物件](#2-別在迴圈體內修改被迭代物件)
* [總結](#總結)
* [系列其他文章](#系列其他文章)
* [註解](#註解)


### 當我們談論容器時，我們在談些什麼？

我在前面給了“容器”一個簡單的定義：*專門用來裝其他物件的就是容器*。但這個定義太寬泛了，無法對我們的日常程式設計產生什麼指導價值。要真正掌握 Python 裡的容器，需要分別從兩個層面入手：

- **底層實現**：內建容器型別使用了什麼資料結構？某項操作如何工作？
- **高層抽象**：什麼決定了某個物件是不是容器？哪些行為定義了容器？

下面，讓我們一起站在這兩個不同的層面上，重新認識容器。


## 底層看容器

Python 是一門高階程式語言，**它所提供的內建容器型別，都是經過高度封裝和抽象後的結果**。和“連結串列”、“紅黑樹”、“雜湊表”這些名字相比，所有 Python 內建型別的名字，都只描述了這個型別的功能特點，其他人完全沒法只通過這些名字瞭解它們的哪怕一丁點內部細節。

這是 Python 程式語言的優勢之一。相比 C 語言這類更接近計算機底層的程式語言，Python 重新設計並實現了對程式設計者更友好的內建容器型別，遮蔽掉了記憶體管理等額外工作。為我們提供了更好的開發體驗。

但如果這是 Python 語言的優勢的話，為什麼我們還要費勁去了解容器型別的實現細節呢？答案是：**關注細節可以幫助我們編寫出更快的程式碼。**

### 寫更快的程式碼

#### 1. 避免頻繁擴充列表/建立新列表

所有的內建容器型別都不限制容量。如果你願意，你可以把遞增的數字不斷塞進一個空列表，最終撐爆整臺機器的記憶體。

在 Python 語言的實現細節裡，列表的記憶體是按需分配的[[注1]](#annot1)，當某個列表當前擁有的記憶體不夠時，便會觸發記憶體擴容邏輯。而分配記憶體是一項昂貴的操作。雖然大部分情況下，它不會對你的程式效能產生什麼嚴重的影響。但是當你處理的資料量特別大時，很容易因為記憶體分配拖累整個程式的效能。

還好，Python 早就意識到了這個問題，並提供了官方的問題解決指引，那就是：**“變懶”**。

如何解釋“變懶”？`range()` 函式的進化是一個非常好的例子。

在 Python 2 中，如果你呼叫 `range(100000000)`，需要等待好幾秒才能拿到結果，因為它需要返回一個巨大的列表，花費了非常多的時間在記憶體分配與計算上。但在 Python 3 中，同樣的呼叫馬上就能拿到結果。因為函式返回的不再是列表，而是一個型別為 `range` 的懶惰物件，只有在你迭代它、或是對它進行切片時，它才會返回真正的數字給你。

**所以說，為了提高效能，內建函式 `range` “變懶”了。** 而為了避免過於頻繁的記憶體分配，在日常編碼中，我們的函式同樣也需要變懶，這包括：

- 更多的使用 `yield` 關鍵字，返回生成器物件
- 儘量使用生成器表示式替代列表推導表示式
    - 生成器表示式：`(i for i in range(100))` 👍
    - 列表推導表示式：`[i for i in range(100)]`
- 儘量使用模組提供的懶惰物件：
    - 使用 `re.finditer` 替代 `re.findall`
    - 直接使用可迭代的檔案物件： `for line in fp`，而不是 `for line in fp.readlines()`

#### 2. 在列表頭部操作多的場景使用 deque 模組

列表是基於陣列結構（Array）實現的，當你在列表的頭部插入新成員（`list.insert(0, item)`）時，它後面的所有其他成員都需要被移動，操作的時間複雜度是 `O(n)`。這導致在列表的頭部插入成員遠比在尾部追加（`list.append(item)` 時間複雜度為 `O(1)`）要慢。

如果你的程式碼需要執行很多次這類操作，請考慮使用 [collections.deque](https://docs.python.org/3.7/library/collections.html#collections.deque) 型別來替代列表。因為 deque 是基於雙端佇列實現的，無論是在頭部還是尾部追加元素，時間複雜度都是 `O(1)`。

#### 3. 使用集合/字典來判斷成員是否存在

當你需要判斷成員是否存在於某個容器時，用集合比列表更合適。因為 `item in [...]` 操作的時間複雜度是 `O(n)`，而 `item in {...}` 的時間複雜度是 `O(1)`。這是因為字典與集合都是基於雜湊表（Hash Table）資料結構實現的。

```python
# 這個例子不是特別恰當，因為當目標集合特別小時，使用集合還是列表對效率的影響微乎其微
# 但這不是重點 :)
VALID_NAMES = ["piglei", "raymond", "bojack", "caroline"]

# 轉換為集合型別專門用於成員判斷
VALID_NAMES_SET = set(VALID_NAMES)


def validate_name(name):
    if name not in VALID_NAMES_SET:
        # 此處使用了 Python 3.6 新增的 f-strings 特性
        raise ValueError(f"{name} is not a valid name!")
```

> Hint: 強烈建議閱讀 [TimeComplexity - Python Wiki](https://wiki.python.org/moin/TimeComplexity)，瞭解更多關於常見容器型別的時間複雜度相關內容。
> 
> 如果你對字典的實現細節感興趣，也強烈建議觀看 Raymond Hettinger 的演講 [Modern Dictionaries(YouTube)](https://www.youtube.com/watch?v=p33CVV29OG8&t=1403s)

## 高層看容器

Python 是一門“[鴨子型別](https://en.wikipedia.org/wiki/Duck_typing)”語言：*“當看到一隻鳥走起來像鴨子、游泳起來像鴨子、叫起來也像鴨子，那麼這隻鳥就可以被稱為鴨子。”* 所以，當我們說某個物件是什麼型別時，在根本上其實指的是： **這個物件滿足了該型別的特定介面規範，可以被當成這個型別來使用。** 而對於所有內建容器型別來說，同樣如此。

開啟位於 [collections](https://docs.python.org/3.7/library/collections.html) 模組下的 [abc](https://docs.python.org/3/library/collections.abc.html)*（“抽象類 Abstract Base Classes”的首字母縮寫）* 子模組，可以找到所有與容器相關的介面（抽象類）[[注2]](#annot2)定義。讓我們分別看看那些內建容器型別都滿足了什麼介面：

- **列表（list）**：滿足 `Iterable`、`Sequence`、`MutableSequence` 等介面
- **元組（tuple）**：滿足 `Iterable`、`Sequence`
- **字典（dict）**：滿足 `Iterable`、`Mapping`、`MutableMapping` [[注3]](#annot3)
- **集合（set）**：滿足 `Iterable`、`Set`、`MutableSet` [[注4]](#annot4)

每個內建容器型別，其實就是滿足了多個介面定義的組合實體。比如所有的容器型別都滿足 `“可被迭代的”（Iterable`） 這個介面，這意味著它們都是“可被迭代”的。但是反過來，不是所有“可被迭代”的物件都是容器。就像字串雖然可以被迭代，但我們通常不會把它當做“容器”來看待。

瞭解這個事實後，我們將**在 Python 裡重新認識**面向物件程式設計中最重要的原則之一：**面向介面而非具體實現來程式設計。**

讓我們透過一個例子，看看如何理解 Python 裡的“面向介面程式設計”。

### 寫擴充套件性更好的程式碼

某日，我們接到一個需求：*有一個列表，裡面裝著很多使用者評論，為了在頁面正常展示，需要將所有超過一定長度的評論用省略號替代*。

這個需求很好做，很快我們就寫出了第一個版本的程式碼：

```python
# 注：為了加強示例程式碼的說明性，本文中的部分程式碼片段使用了Python 3.5
# 版本新增的 Type Hinting 特性

def add_ellipsis(comments: typing.List[str], max_length: int = 12):
    """如果評論列表裡的內容超過 max_length，剩下的字元用省略號代替
    """
    index = 0
    for comment in comments:
        comment = comment.strip()
        if len(comment) > max_length:
            comments[index] = comment[:max_length] + '...'
        index += 1
    return comments


comments = [
    "Implementation note",
    "Changed",
    "ABC for generator",
]
print("\n".join(add_ellipsis(comments)))
# OUTPUT:
# Implementati...
# Changed
# ABC for gene...
```

上面的程式碼裡，`add_ellipsis` 函式接收一個列表作為引數，然後遍歷它，替換掉需要修改的成員。這一切看上去很合理，因為我們接到的最原始需求就是：“有一個 **列表**，裡面...”。**但如果有一天，我們拿到的評論不再是被繼續裝在列表裡，而是在不可變的元組裡呢？**

那樣的話，現有的函式設計就會逼迫我們寫出 `add_ellipsis(list(comments))` 這種即慢又難看的程式碼了。😨

#### 面向容器介面程式設計

我們需要改進函式來避免這個問題。因為 `add_ellipsis` 函式強依賴了列表型別，所以當引數型別變為元組時，現在的函式就不再適用了*（原因：給 `comments[index]` 賦值的地方會丟擲 `TypeError` 異常）。* 如何改善這部分的設計？祕訣就是：**讓函式依賴“可迭代物件”這個抽象概念，而非實體列表型別。**

使用生成器特性，函式可以被改成這樣：

```python
def add_ellipsis_gen(comments: typing.Iterable[str], max_length: int = 12):
    """如果可迭代評論裡的內容超過 max_length，剩下的字元用省略號代替
    """
    for comment in comments:
        comment = comment.strip()
        if len(comment) > max_length:
            yield comment[:max_length] + '...'
        else:
            yield comment


print("\n".join(add_ellipsis_gen(comments)))
```

在新函式裡，我們將依賴的引數型別從列表改成了可迭代的抽象類。這樣做有很多好處，一個最明顯的就是：無論評論是來自列表、元組或是某個檔案，新函式都可以輕鬆滿足：

```python
# 處理放在元組裡的評論
comments = ("Implementation note", "Changed", "ABC for generator")
print("\n".join(add_ellipsis_gen(comments)))

# 處理放在檔案裡的評論
with open("comments") as fp:
    for comment in add_ellipsis_gen(fp):
        print(comment)
```

將依賴由某個具體的容器型別改為抽象介面後，函式的適用面變得更廣了。除此之外，新函式在執行效率等方面也都更有優勢。現在讓我們再回到之前的問題。**從高層來看，什麼定義了容器？**

答案是： **各個容器型別實現的介面協議定義了容器。** 不同的容器型別在我們的眼裡，應該是 `是否可以迭代`、`是否可以修改`、`有沒有長度` 等各種特性的組合。我們需要在編寫相關程式碼時，**更多的關注容器的抽象屬性，而非容器型別本身**，這樣可以幫助我們寫出更優雅、擴充套件性更好的程式碼。

> Hint：在 [itertools](https://docs.python.org/3/library/itertools.html) 與 [more-itertools](https://pypi.org/project/more-itertools/) 模組裡可以找到更多關於處理可迭代物件的寶藏。

## 常用技巧

### 1. 使用元組改善分支程式碼

有時，我們的程式碼裡會出現超過三個分支的 `if/else` 。就像下面這樣：

```python
import time


def from_now(ts):
    """接收一個過去的時間戳，返回距離當前時間的相對時間文字描述
    """
    now = time.time()
    seconds_delta = int(now - ts)
    if seconds_delta < 1:
        return "less than 1 second ago"
    elif seconds_delta < 60:
        return "{} seconds ago".format(seconds_delta)
    elif seconds_delta < 3600:
        return "{} minutes ago".format(seconds_delta // 60)
    elif seconds_delta < 3600 * 24:
        return "{} hours ago".format(seconds_delta // 3600)
    else:
        return "{} days ago".format(seconds_delta // (3600 * 24))


now = time.time()
print(from_now(now))
print(from_now(now - 24))
print(from_now(now - 600))
print(from_now(now - 7500))
print(from_now(now - 87500))
# OUTPUT:
# less than 1 second ago
# 24 seconds ago
# 10 minutes ago
# 2 hours ago
# 1 days ago
```

上面這個函式挑不出太多毛病，很多很多人都會寫出類似的程式碼。但是，如果你仔細觀察它，可以在分支程式碼部分找到一些明顯的“**邊界**”。 比如，當函式判斷某個時間是否應該用“秒數”展示時，用到了 `60`。而判斷是否應該用分鐘時，用到了 `3600`。

**從邊界提煉規律是最佳化這段程式碼的關鍵。** 如果我們將所有的這些邊界放在一個有序元組中，然後配合二分查詢模組 [bisect](https://docs.python.org/3.7/library/bisect.html)。整個函式的控制流就能被大大簡化：

```python
import bisect


# BREAKPOINTS 必須是已經排好序的，不然無法進行二分查詢
BREAKPOINTS = (1, 60, 3600, 3600 * 24)
TMPLS = (
    # unit, template
    (1, "less than 1 second ago"),
    (1, "{units} seconds ago"),
    (60, "{units} minutes ago"),
    (3600, "{units} hours ago"),
    (3600 * 24, "{units} days ago"),
)


def from_now(ts):
    """接收一個過去的時間戳，返回距離當前時間的相對時間文字描述
    """
    seconds_delta = int(time.time() - ts)
    unit, tmpl = TMPLS[bisect.bisect(BREAKPOINTS, seconds_delta)]
    return tmpl.format(units=seconds_delta // unit)
```

除了用元組可以最佳化過多的 `if/else` 分支外，有些情況下字典也能被用來做同樣的事情。關鍵在於從現有程式碼找到重複的邏輯與規律，並多多嘗試。

### 2. 在更多地方使用動態解包

動態解包操作是指使用 `*` 或 `**` 運算子將可迭代物件“解開”的行為，在 Python 2 時代，這個操作只能被用在函式引數部分，並且對出現順序和數量都有非常嚴格的要求，使用場景非常單一。

```python
def calc(a, b, multiplier=1):
    return (a + b) * multiplier


# Python2 中只支援在函式引數部分進行動態解包
print calc(*[1, 2], **{"multiplier": 10})
# OUTPUT: 30
```

不過，Python 3 尤其是 3.5 版本後，`*` 和 `**` 的使用場景被大大擴充了。舉個例子，在 Python 2 中，如果我們需要合併兩個字典，需要這麼做：

```python
def merge_dict(d1, d2):
    # 因為字典是可被修改的物件，為了避免修改原物件，此處需要複製一個 d1 的淺複製
    result = d1.copy()
    result.update(d2)
    return result
    
user = merge_dict({"name": "piglei"}, {"movies": ["Fight Club"]})
```

但是在 Python 3.5 以後的版本，你可以直接用 `**` 運算子來快速完成字典的合併操作：

```
user = {**{"name": "piglei"}, **{"movies": ["Fight Club"]}}
```

除此之外，你還可以在普通賦值語句中使用 `*` 運算子來動態的解包可迭代物件。如果你想詳細瞭解相關內容，可以閱讀下面推薦的 PEP。

> Hint：推進動態解包場景擴充的兩個 PEP：
> - [PEP 3132 -- Extended Iterable Unpacking | Python.org](https://www.python.org/dev/peps/pep-3132/)
> - [PEP 448 -- Additional Unpacking Generalizations | Python.org](https://www.python.org/dev/peps/pep-0448/)

### 3. 使用 next() 函式

`next()` 是一個非常實用的內建函式，它接收一個迭代器作為引數，然後返回該迭代器的下一個元素。使用它配合生成器表示式，可以高效的實現*“從列表中查詢第一個滿足條件的成員”* 之類的需求。

```python
numbers = [3, 7, 8, 2, 21]
# 獲取並 **立即返回** 列表裡的第一個偶數
print(next(i for i in numbers if i % 2 == 0))
# OUTPUT: 8
```

### 4. 使用有序字典來去重

字典和集合的結構特點保證了它們的成員不會重複，所以它們經常被用來去重。但是，使用它們倆去重後的結果會丟失原有列表的順序。這是由底層資料結構“雜湊表（Hash Table）”的特點決定的。

```python
>>> l = [10, 2, 3, 21, 10, 3]
# 去重但是丟失了順序
>>> set(l)
{3, 10, 2, 21}
```

如果既需要去重又必須保留順序怎麼辦？我們可以使用 `collections.OrderedDict` 模組:

```python
>>> from collections import OrderedDict
>>> list(OrderedDict.fromkeys(l).keys())
[10, 2, 3, 21]
```

> Hint: 在 Python 3.6 中，預設的字典型別修改了實現方式，已經變成有序的了。並且在 Python 3.7 中，該功能已經從 **語言的實現細節** 變成了為 **可依賴的正式語言特性**。
> 
> 但是我覺得讓整個 Python 社群習慣這一點還需要一些時間，畢竟目前“字典是無序的”還是被印在無數本 Python 書上。所以，我仍然建議在一切需要有序字典的地方使用 OrderedDict。

## 常見誤區

### 1. 當心那些已經枯竭的迭代器

在文章前面，我們提到了使用“懶惰”生成器的種種好處。但是，所有事物都有它的兩面性。生成器的最大的缺點之一就是：**它會枯竭**。當你完整遍歷過它們後，之後的重複遍歷就不能拿到任何新內容了。

```python
numbers = [1, 2, 3]
numbers = (i * 2 for i in numbers)

# 第一次迴圈會輸出 2, 4, 6
for number in numbers:
    print(number)

# 這次迴圈什麼都不會輸出，因為迭代器已經枯竭了
for number in numbers:
    print(number)
```

而且不光是生成器表示式，Python 3 裡的 map、filter 內建函式也都有一樣的特點。忽視這個特點很容易導致程式碼中出現一些難以察覺的 Bug。

Instagram 就在專案從 Python 2 到 Python 3 的遷移過程中碰到了這個問題。它們在 PyCon 2017 上分享了對付這個問題的故事。訪問文章 [Instagram 在 PyCon 2017 的演講摘要](https://www.zlovezl.cn/articles/instagram-pycon-2017/)，搜尋“迭代器”可以檢視詳細內容。

### 2. 別在迴圈體內修改被迭代物件

這是一個很多 Python 初學者會犯的錯誤。比如，我們需要一個函式來刪掉列表裡的所有偶數：

```python
def remove_even(numbers):
    """去掉列表裡所有的偶數
    """
    for i, number in enumerate(numbers):
        if number % 2 == 0:
            # 有問題的程式碼
            del numbers[i]


numbers = [1, 2, 7, 4, 8, 11]
remove_even(numbers)
print(numbers)
# OUTPUT: [1, 7, 8, 11]
```

注意到結果裡那個多出來的 “8” 了嗎？當你在遍歷一個列表的同時修改它，就會出現這樣的事情。因為被迭代的物件 `numbers` 在迴圈過程中被修改了。**遍歷的下標在不斷增長，而列表本身的長度同時又在不斷縮減。這樣就會導致列表裡的一些成員其實根本就沒有被遍歷到。**

所以對於這類操作，請使用一個新的空列表儲存結果，或者利用 `yield` 返回一個生成器。而不是修改被迭代的列表或是字典物件本身。

## 總結

在這篇文章中，我們首先從“容器型別”的定義出發，在底層和高層兩個層面探討了容器型別。之後遵循系列文章傳統，提供了一些編寫容器相關程式碼時的技巧。

讓我們最後再總結一下要點：

- 瞭解容器型別的底層實現，可以幫助你寫出效能更好的程式碼
- 提煉需求裡的抽象概念，面向介面而非實現程式設計
- 多使用“懶惰”的物件，少生成“迫切”的列表
- 使用元組和字典可以簡化分支程式碼結構
- 使用 `next()` 函式配合迭代器可以高效完成很多事情，但是也需要注意“枯竭”問題
- collections、itertools 模組裡有非常多有用的工具，快去看看吧！

看完文章的你，有沒有什麼想吐槽的？請留言或者在 [專案 Github Issues](https://github.com/piglei/one-python-craftsman) 告訴我吧。

[>>>下一篇【5.讓函式返回結果的技巧】](5-function-returning-tips.md)

[<<<上一篇【3.使用數字與字串的技巧】](3-tips-on-numbers-and-strings.md)

## 系列其他文章

- [所有文章索引 [Github]](https://github.com/piglei/one-python-craftsman)
- [Python 工匠：善用變數改善程式碼質量](https://www.zlovezl.cn/articles/python-using-variables-well/)
- [Python 工匠：編寫條件分支程式碼的技巧](https://www.zlovezl.cn/articles/python-else-block-secrets/)
- [Python 工匠：使用數字與字串的技巧](https://www.zlovezl.cn/articles/tips-on-numbers-and-strings/)

## 註解

1. <a id="annot1"></a>Python 這門語言除了 CPython 外，還有許許多多的其他版本實現。如無特別說明，本文以及 “Python 工匠” 系列裡出現的所有 Python 都特指 Python 的 C 語言實現 CPython
2. <a id="annot2"></a>Python 裡沒有類似其他程式語言裡的“Interface 介面”型別，只有類似的“抽象類”概念。為了表達方便，後面的內容均統一使用“介面”來替代“抽象類”。
3. <a id="annot3"></a>有沒有隻實現了 Mapping 但又不是 MutableMapping 的型別？試試 [MappingProxyType({})](https://docs.python.org/3/library/types.html#types.MappingProxyType)
4. <a id="annot4"></a>有沒有隻實現了 Set 但又不是 MutableSet 的型別？試試 [frozenset()](https://docs.python.org/3/library/stdtypes.html#frozenset)

