---
title: kotlinで24時以降の時刻文字列をいい感じに扱う
tags:
  - Java
  - Kotlin
  - 日付
private: false
updated_at: '2018-10-22T16:52:25+09:00'
id: ab6e0bd88dabfce2ac66
organization_url_name: null
---


UIから"HH:mm"形式の文字列を受け取り、処理日の日付を足してDate型にする、という処理を書きました。

例：10月20日に処理を実行する場合

|受け取る文字列|返すDateの日時|
|-----------------|------------------|
| "10:00"|2018/10/20 10:00:00|
| "20:00"|2018/10/20 20:00:00|
| "30:00"|2018/10/21 06:00:00|
| "40:00"|2018/10/21 16:00:00|

やっかいだったのが、24時以降の日付を受け取った場合は1日後の日付とする必要があったことです。

以下、実装してみたメソッドです。
（翌々日以降の日時が来ることはない仕様でしたが、実装上は99時まで対応してます。）


```kotlin
    /**
     * "HH:mm"形式の文字列を、当日の日付を付与したDate型で返却する.
     * 24時以降が指定された場合、対応する翌日以降の日付を返却する.
     * @param "HH:mm"形式の文字列.
     * @return Date型、形式が不正ならnull
     */
    fun String.toTodayDate(): Date? {
        // 形式が不正ならnull返却
        val split = this.split(':')
        if (split.size != 2) return null
        val hour = split[0].toIntOrNull() ?: return null
        val minute = split[1].toIntOrNull() ?: return null

        val c = Calendar.getInstance()
        // 指定された時・分をset
        // 24を超えても正規化される
        c.set(Calendar.HOUR_OF_DAY, hour)
        c.set(Calendar.MINUTE, minute)
        //秒以下は0を指定
        c.set(Calendar.SECOND, 0)
        c.set(Calendar.MILLISECOND, 0)

        return c.time
    }
```

Stringの拡張関数として実装しています。
24時以降の文字列の処理の方法がわからずに微妙にハマったのですが、
Calendarのset()は範囲外の数値でもいい感じにしてくれるとのことで、すんなりこれで実装できました。
c.set()が続くのが冗長ですが、これはそういうもんなんですかね。

---

もっとスマートな書き方があれば、ご教授いただければ助かります。
また、Kotlin初心者なので、文法面・体裁面の指摘もお待ちしてます。
