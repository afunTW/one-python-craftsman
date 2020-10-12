# Python 工匠：一個關於模組的小故事

## 前言

> 這是 “Python 工匠”系列的第 9 篇文章。[[檢視系列所有文章]](https://github.com/piglei/one-python-craftsman)

<div style="text-align: center; color: #999; margin: 14px 0 14px;font-size: 12px;">
<img src="https://www.zlovezl.cn/static/uploaded/2019/05/ricardo-gomez-angel-669574-unsplash_w1280.jpg" width="100%" />
</div>

模組（Module）是我們用來組織 Python 程式碼的基本單位。很多功能強大的複雜站點，都由成百上千個獨立模組共同組成。

雖然模組有著不可替代的用處，但它有時也會給我們帶來麻煩。比如，當你接手一個新專案後，剛展開專案目錄。第一眼就看到了攀枝錯節、難以理解的模組結構，那你肯定會想： *“這專案也太難搞了。”* 😂

在這篇文章裡，我準備了一個和模組有關的小故事與你分享。

## 一個關於模組的小故事

小 R 是一個剛從學校畢業的計算機專業學生。半個月前，他面試進了一家網際網路公司做 Python 開發，負責一個與使用者活動積分有關的小專案。專案的主要功能是查詢站點活躍使用者，併為他們傳送有關活動積分的通知： *“親愛的使用者，您好，您當前的活動積分為 x”*。

專案主要由 `notify_users.py` 指令碼和 `fancy_site` 包組成，結構與各檔案內容如下：

```text
├── fancy_site
│   ├── __init__.py
│   ├── marketing.py        # 與市場活動有關的內容
│   └── users.py            # 與使用者有關的內容
└── notify_users.py     # 指令碼：傳送積分通知
```

檔案 `notify_users.py`：

```python
from fancy_site.users import list_active_users
from fancy_site.marketing import query_user_points


def main():
    """獲取所有的活躍使用者，將積分情況傳送給他們"""
    users = get_active_users()
    points = list_user_points(users)
    for user in users:
        user.add_notification(... ...)
        #  <... 已省略 ...>
```

檔案 `fancy_site/users.py`：

```python
from typing import List


class User:
    # <... 已省略 ...>

    def add_notification(self, message: str):
        """為使用者傳送新通知"""
        pass


def list_active_users() -> List[User]:
    """查詢所有活躍使用者"""
    pass
```

檔案：`fancy_site/marketing.py`：

```python
from typing import List
from .users import User


def query_user_points(users: List[User]) -> List[int]:
    """批次查詢使用者活動積分"""


def send_sms(phone_number: str, message: str):
    """為某手機號傳送簡訊"""
```

只要在專案目錄下執行 `python notify_user.py`，就能實現給所有活躍使用者傳送通知。

### 需求變更

但有一天，產品經理找過來說，光給使用者發站內信通知還不夠，容易被使用者忽略。除了站內信以外，我們還需要同時給使用者推送一條簡訊通知。

琢磨了五秒鐘後，小 R 跟產品經理說：*“這個需求可以做！”*。畢竟給手機號傳送簡訊的 `send_sms()` 函式早就已經有人寫好了。他只要先給 `add_notification` 方法新增一個可選引數 `enable_sms=False`，當傳值為 `True` 時呼叫 `fancy_site.marketing` 模組裡的 `send_sms` 函式就行。

一切聽上去根本沒有什麼難度可言，十分鐘後，小 R 就把 `user.py` 改成了下面這樣：

```python
# 匯入 send_sms 模組的傳送簡訊函式
from .marketing import send_sms


class User:
    # <...> 相關初始化程式碼已省略

    def add_notification(self, message: str, enable_sms=False):
        """為使用者新增新通知"""
        if enable_sms:
            send_sms(user.mobile_number, ... ...)
```

但是，當他修改完程式碼，再次執行 `notify_users.py` 指令碼時，程式卻報錯了：

```raw
Traceback (most recent call last):
  File "notify_users.py", line 2, in <module>
    from fancy_site.users import list_active_users
  File .../fancy_site/users.py", line 3, in <module>
    from .marketing import send_sms
  File ".../fancy_site/marketing.py", line 3, in <module>
    from .users import User
ImportError: cannot import name 'User' from 'fancy_site.users' (.../fancy_site/users.py)
```

錯誤資訊說，無法從 `fancy_site.users` 模組匯入 `User` 物件。

### 解決環形依賴問題

小 R 仔細分析了一下錯誤，發現錯誤是因為 `users` 與 `marketing` 模組之間產生的環形依賴關係導致的。

當程式在 `notify_users.py` 檔案匯入 `fancy_site.users` 模組時，`users` 模組發現自己需要從 `marketing` 模組那裡匯入 `send_sms` 函式。而直譯器在載入 `marketing` 模組的過程中，又反過來發現自己需要依賴 `users` 模組裡面的 `User` 物件。

如此一來，整個模組依賴關係成為了環狀，程式自然也就沒法執行下去了。

![modules_before](https://www.zlovezl.cn/static/uploaded/2019/05/modules_before.png)

不過，沒有什麼問題能夠難倒一個可以正常訪問 Google 的程式設計師。小 R 隨便上網一搜，發現這樣的問題很好解決。因為 Python 的 import 語句非常靈活，他只需要 **把在 users 模組內匯入 send_sms 函式的語句挪到 `add_notification` 方法內，延緩 import 語句的執行就行啦。**

```python
class User:
    # <...> 相關初始化程式碼已省略

    def add_notification(self, message: str, send_sms=False):
        """為使用者新增新通知"""
        # 延緩 import 語句執行
        from .marketing import send_sms
```

改動一行程式碼後，大功告成。小 R 簡單測試後，發現一切正常，然後把程式碼推送了上去。不過小 R 還沒來得及為自己點個贊，意料之外的事情發生了。

這段明明幾乎完美的程式碼改動在 **Code Review** 的時候被審計人小 C 拒絕了。

### 小 C 的疑問

小 R 的同事小 C 是一名有著多年經驗的 Python 程式設計師，他對小 R 說：“使用延遲 import，雖然可以馬上解決包匯入問題。但這個小問題背後隱藏了更多的資訊。比如，**你有沒有想過 send_sms 函式，是不是已經不適合放在 marketing 模組裡了？”**

被小 C 這麼一問，聰明的小 R 馬上意識到了問題所在。要在 `users` 模組內傳送簡訊，重點不在於用延遲匯入解決環形依賴。而是要以此為契機，**發現當前模組間依賴關係的不合理，拆分/合併模組，建立新的分層與抽象，最終消除環形依賴。**

認識清楚問題後，他很快提交了新的程式碼修改。在新程式碼中，他建立了一個專門負責通知與訊息類的工具模組 `msg_utils`，然後把 `send_sms` 函式挪到了裡面。之後 `users` 模組內就可以毫無困難的從 `msg_utils` 模組中匯入 `send_sms` 函數了。

```python
from .msg_utils import send_sms
```

新的模組依賴關係如下圖所示：

![modules_afte](https://www.zlovezl.cn/static/uploaded/2019/05/modules_after.png)

在新的模組結構中，整個專案被整齊的分為三層，模組間的依賴關係也變得只有**單向流動**。之前在函式內部 `import` 的“延遲匯入”技巧，自然也就沒有用武之地了。

小 R 修改後的程式碼獲得了大家的認可，很快就被合併到了主分支。故事暫告一段落，那麼這個故事告訴了我們什麼道理呢？

## 總結

模組間的迴圈依賴是一個在大型 Python 專案中很常見的問題，越複雜的專案越容易碰到這個問題。當我們在參與這些專案時，**如果對模組結構、分層、抽象缺少應有的重視。那麼專案很容易就會慢慢變得複雜無比、難以維護。**

所以，合理的模組結構與分層非常重要。它可以大大降低開發人員的心智負擔和專案維護成本。這也是我為什麼要和你分享這個簡單故事的原因。“在函式內延遲 import” 的做法當然沒有錯，但我們更應該關注的是：**整個專案內的模組依賴關係與分層是否合理。**

最後，讓我們再嘗試從 小 R 的故事裡強行總結出幾個道理吧：

- 合理的模組結構與分層可以降低專案的開發維護成本
- 合理的模組結構不是一成不變的，應該隨著專案發展調整
- 遇到問題時，不要選**“簡單但有缺陷”**的那個方案，要選**“麻煩但正確”**的那個
- 整個專案內的模組間依賴關係流向，應該是單向的，不能有環形依賴存在

看完文章的你，有沒有什麼想吐槽的？請留言或者在 [專案 Github Issues](https://github.com/piglei/one-python-craftsman) 告訴我吧。

[>>>下一篇【10.做一個精通規則的玩家】](10-a-good-player-know-the-rules.md)

[<<<上一篇【8.使用裝飾器的技巧】](8-tips-on-decorators.md)

## 附錄

- 題圖來源: Photo by Ricardo Gomez Angel on Unsplash
- 更多系列文章地址：https://github.com/piglei/one-python-craftsman

系列其他文章：

- [所有文章索引 [Github]](https://github.com/piglei/one-python-craftsman)
- [Python 工匠：編寫條件分支程式碼的技巧](https://www.zlovezl.cn/articles/python-else-block-secrets/)
- [Python 工匠：異常處理的三個好習慣](https://www.zlovezl.cn/articles/three-rituals-of-exceptions-handling/)
- [Python 工匠：編寫地道迴圈的兩個建議](https://www.zlovezl.cn/articles/two-tips-on-loop-writing/)



