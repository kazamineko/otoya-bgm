<template>
  <div class="background-container">
    <!-- ここに今後、半透明のパネルやメニューボタンなどを追加していきます -->
  </div>
</template>

<style>
body, html {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  font-family: sans-serif; /* 今後のフォント設定のための仮設定 */
}

.background-container {
  /* 背景画像の設定 */
  background-image: url('/bg-main.jpg'); /* publicフォルダに置いた画像を指定 */
  background-size: cover; /* 画面に合わせて画像を拡大・縮小 */
  background-position: center; /* 画像の中央を基準に表示 */
  background-repeat: no-repeat; /* 画像を繰り返さない */
  
  /* 画面全体に広げる設定 */
  width: 100vw; /* 画面の横幅いっぱいに */
  height: 100vh; /* 画面の高さいっぱいに */
  
  /* 今後のコンテンツ配置のための設定 */
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>