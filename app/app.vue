<template>
  <div class="background-container">
    
    <!-- ここからが追加したパネル部分 -->
    <div class="content-panel">
      <!-- 今後ここに「おとや」のタイトルやメニューボタンが入ります -->
      <h1 style="color: #4a4a4a;">（仮）AI-BGM 喫茶「おとや」</h1>
    </div>
    <!-- ここまで -->

  </div>
</template>

<style>
body, html {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  font-family: sans-serif;
}

.background-container {
  background-image: url('/bg-main.jpg');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  width: 100vw;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

/* ▼▼▼ ここからが追加したスタイル ▼▼▼ */
.content-panel {
  /* パネルの見た目 */
  background-color: rgba(255, 255, 255, 0.8); /* 背景を白の80%透明に */
  padding: 40px; /* 内側の余白 */
  border-radius: 8px; /* 角を少し丸くする */
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2); /* 影を付けて立体感を出す */

  /* 和紙のような質感（オプション） */
  backdrop-filter: blur(5px); /* 背景を少しぼかす（モダンなブラウザのみ対応） */

  /* サイズと配置 */
  width: 90%; /* 横幅 */
  max-width: 600px; /* PCで見たときに広がりすぎないように */
  text-align: center; /*中の文字を中央揃えに*/
}
/* ▲▲▲ ここまで ▲▲▲ */
</style>