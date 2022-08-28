<script setup>
import parseAPNG from "apng-js";
import { fabric } from "fabric";
import { onMounted } from "vue";

const canvasWidth = (window.innerWidth / 3); //Canvasの横の長さ
const canvasHeight = (window.innerHeight / 2); //Canvasの縦の長さ
var apng;
var video;
var player = null;
var canvasVideo;
var ctxVideo;
var canvasPhotoframe;
var ctxPhotoframe;
var canvasStamp;
var ctxStamp;
var canvasStampFab;
var canvasConcat;
var ctxConcat;
let cameraFacing = false;

onMounted(() => {
  //canvasのdomとctx
  //video
  canvasVideo = document.getElementById("canvasVideo");
  ctxVideo = canvasVideo.getContext('2d');
  //photoframe
  canvasPhotoframe = document.getElementById('canvasPhotoframe');
  ctxPhotoframe = canvasPhotoframe.getContext('2d');
  //stamp
  canvasStamp = document.querySelector("#canvasStamp");
  ctxStamp = canvasStamp.getContext('2d');
  canvasStampFab = new fabric.Canvas("canvasStamp");
  //concat
  canvasConcat = document.getElementById("canvasConcat");
  ctxConcat = canvasConcat.getContext("2d");

  //ウェブカメラ映像をvideoに描写（インビジブル）
  video = document.getElementById("video")
  navigator.mediaDevices.getUserMedia({
    video: { facingMode: "environment" },
    audio: false,
  }).then(stream => {
    video.srcObject = stream;
    video.play()
  }).catch(e => {
    console.log(e)
  })
  // canvasにvideo要素の映像をcanvasに描画する
  function _canvasUpdate() {
    ctxVideo.drawImage(video, 0, 0, canvasWidth, canvasHeight);
    requestAnimationFrame(_canvasUpdate);
  };
  _canvasUpdate();
})
//cannvas画像を合成してcaptureに描画
const concatImage = () => {
  //fabricのアクティブを解除
  canvasStampFab.discardActiveObject();
  canvasStampFab.renderAll();
  //3つのcanvasをconcatcanvasに順次描画
  let canvasList = [canvasVideo, canvasPhotoframe, canvasStamp];
  for (let i = 0; i < canvasList.length; ++i) {
    ctxConcat.drawImage(canvasList[i], 0, 0, canvasWidth, canvasHeight);
  }
}
//pngファイル(photoframe)を読み込みcanvasに描画
const fileSelected = (event) => {
  var file = event.target.files[0];
  var reader = new FileReader();
  reader.onload = () => {
    apng = parseAPNG(reader.result);;
    if (apng instanceof Error) {
      // handle error
    } else {
      apng.getPlayer(ctxPhotoframe, true);
    }
  }
  reader.readAsArrayBuffer(file);
};
//capture画像をダウンロード
const download = () => {
  const linkConcat = document.createElement('a');
  linkConcat.href = canvasConcat.toDataURL();
  linkConcat.download = 'export_image_wrap.png';
  linkConcat.click();

};
//stopボタン：現在設定なし
const frameStop = () => {
};
//スタンプ：canvasStamp 
const onStamp = () => {
  const imgStamp = new Image();
  imgStamp.onload = function () {
    //image
    fabric.Image.fromURL(imgStamp.src, function (oImg) {
      oImg.scale(1);
      canvasStampFab.add(oImg);
      oImg.hasBorders = true;//枠線をなくす
      oImg.hasControls = true;//枠線の□をなくす
    });
  }
  imgStamp.src = "/supplepentan-favicon.png"
};
// カメラ切り替え
const changeCamera = (event) => {
  event.preventDefault();
  let video = document.getElementById("video")
  let mode = cameraFacing ? "environment" : "user";
  // Android Chromeでは、セッションを一時停止しないとエラーが出ることがある
  stopStreamedVideo(video);
  navigator.mediaDevices.getUserMedia({ video: { facingMode: mode } })
    .then(stream => video.srcObject = stream)
  // .catch(err => alert(`${err.name} ${err.message}`));
  cameraFacing = !cameraFacing;
};
const stopStreamedVideo = (video) => {
  let stream = video.srcObject;
  let tracks = stream.getTracks();
  tracks.forEach(function (track) {
    track.stop();
  });
  video.srcObject = null;
};
</script>
<template>
  <div>
    <div id="containerCanvas" class="grid grid-cols-2 py-2">
      <div class="">
        <h1 class="text-center">Real Time</h1>
        <div class="relative py-2 border-2 border-sky-200 flex justify-center">
          <canvas v-bind:width="canvasWidth" v-bind:height="canvasHeight" class="absolute classCanvas"
            id="canvasVideo"></canvas>
          <canvas v-bind:width="canvasWidth" v-bind:height="canvasHeight" class="absolute classCanvas"
            id="canvasPhotoframe" v-on:mousemove="position"></canvas>
          <canvas v-bind:width="canvasWidth" v-bind:height="canvasHeight" class="absolute classCanvas"
            id="canvasStamp"></canvas>
        </div>
      </div>
      <div>
        <h1 class="text-center">Capture</h1>
        <div class="py-2 border-2 border-sky-200 flex justify-center">
          <canvas v-bind:width="canvasWidth" v-bind:height="canvasHeight" class="classCanvas"
            id="canvasConcat"></canvas>
        </div>
      </div>
    </div>
  </div>
  <div class="grid grid-cols-2 border-2">
    <div class="p-2 border-2 rounded border-sky-200">
      <p class="text-center">Photoframe</p>
      <input type="file" v-on:change="fileSelected">
    </div>
    <div class="p-2 border-2 rounded border-sky-200">
      <p class="text-center border">Stamp</p>
      <div>
        <img src="/supplepentan-favicon.png" alt="" v-on:click="onStamp">
      </div>
    </div>
  </div>
  <div class="flex items-stretch">
    <button type="button" class="flex-1 classButton" v-on:click="concatImage">
      <p>Capture</p>
    </button>
    <button type="button" class="flex-1 classButton" v-on:click="download">
      <p>Download</p>
    </button>
    <button type="button" class="flex-1 classButton" v-on:click="frameStop">
      <p>STOP</p>
    </button>
    <button type="button" class="flex-1 classButton" v-on:click="changeCamera">
      <p>changeCamera</p>
    </button>
  </div>
  <div class="invisible">
    <video id="video" autoplay playsinline="true"></video>
  </div>
</template>
<style scoped>
.classCanvas {
  @apply rounded
}

.classButton {
  @apply block p-2 rounded bg-neutral-200 border-2 border-sky-200;
}
</style>