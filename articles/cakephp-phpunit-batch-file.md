---
title: "[CakePHP4/PHPUnit]バッチファイルのテスト"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["php"]
published: false # 公開設定
---

CakePHP4のバッチ処理用の関数実装時、PHPUnitでテストを作成する際のメモ

## fixtures
`$fixtures`がある場合、ここでテスト用に用意した fixture をロードする。
```php:MessagesCommandTest.php
protected $fixtures = [
    'app.Command/MessagesCommand/Messages',
    'app.Command/MessagesCommand/Users',
    'app.Command/MessagesCommand/Rooms',
];
```

## テスト用関数
```php
// sendMessage()という関数に対する
public function testSendMessage(): void
{
    // fixture のデータに対しバッチを実行する
    $this->exec('Messages');

    // ステータスコード 0 で終了したことを確認
    $this->assertExitCode(Command::CODE_SUCCESS);
    $this->assertExitSuccess(Command::CODE_SUCCESS);
    $this->assertExitError(Command::CODE_SUCCESS);
    $this->assertOutputEmpty();
    $this->assertErrorRegExp();
    $this->
    assertOutputEmpty
    // 実行後、結果を比較するための関数
    $this->assertSame(10, $this->Messages->findByUserId(1)->toList()[0]['id']);
}
```


## 参考文献
https://book.cakephp.org/4/ja/development/testing.html
https://phpunit.readthedocs.io/ja/latest/