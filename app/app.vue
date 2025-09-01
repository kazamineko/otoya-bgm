<template>
  <div class="background-container">
    <div class="content-panel">

      <!-- タイトル -->
      <h1 class="title">AI-BGM 喫茶「おとや」</h1>
      <p class="subtitle">本日のBGMをお選びください</p>

      <!-- メニューボタンコンテナ -->
      <div class="menu-container">
        <!-- 集中ブレンド -->
        <button class="menu-button">
          <span class="menu-title">集中ブレンド</span>
          <span class="menu-description">思考を妨げない、静かな雨音のような音楽。</span>
        </button>
        <!-- リラックス・デカフェ -->
        <button class="menu-button">
          <span class="menu-title">リラックス・デカフェ</span>
          <span class="menu-description">心のコリをほぐす、優しい陽だまりのような音楽。</span>
        </button>
        <!-- ジャズ・スペシャル -->
        <button class="menu-button">
          <span class="menu-title">ジャズ・スペシャル</span>
          <span class="menu-description">夜の静寂に寄り添う、マスターこだわりの一杯。</span>
        </button>
        <!-- Lo-Fi・ビター -->
        <button class="menu-button">
          <span class="menu-title">Lo-Fi・ビター</span>
          <span class="menu-description">懐かしいレコードに針を落とす、あの感覚をあなたに。</span>
        </button>
      </div>

    </div>
  </div>
</template>

<style>
/* ... (body, html, background-container, content-panelのスタイルは変更なし) ... */
body, html { margin: 0; padding: 0; width: 100%; height: 100%; font-family: 'Hiragino Mincho ProN', 'MS Mincho', serif; }
.background-container { background-image: url('/bg-main.jpg'); background-size: cover; background-position: center; background-repeat: no-repeat; width: 100vw; height: 100vh; display: flex; justify-content: center; align-items: center; }
.content-panel { background-color: rgba(255, 255, 255, 0.85); padding: 20px 40px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2); backdrop-filter: blur(5px); width: 90%; max-width: 600px; text-align: center; }

/* ▼▼▼ ここからが今回の変更・追加箇所 ▼▼▼ */
.title {
  color: #363636;
  font-weight: bold;
  margin-bottom: 8px;
}

.subtitle {
  color: #555;
  margin-top: 0;
  margin-bottom: 30px;
}

.menu-container {
  display: flex;
  flex-direction: column; /* ボタンを縦に並べる */
  gap: 15px; /* ボタンとボタンの間隔 */
}

.menu-button {
  background-color: #f5f5f5;
  border: 1px solid #dbdbdb;
  border-radius: 4px;
  padding: 15px 20px;
  cursor: pointer; /* カーソルを乗せると指マークに */
  transition: all 0.2s ease; /* アニメーションを滑らかに */
  text-align: left;
  display: flex;
  flex-direction: column;
  font-family: inherit; /* 親要素のフォントを引き継ぐ */
}

.menu-button:hover {
  background-color: #e8e8e8;
  border-color: #b5b5b5;
  transform: translateY(-2px); /* 少し上に浮き上がる */
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.menu-title {
  font-size: 1.1em;
  font-weight: bold;
  color: #363636;
}

.menu-description {
  font-size: 0.9em;
  color: #7a7a7a;
  margin-top: 4px;
}
/* ▲▲▲ ここまで ▲▲▲ */
</style>