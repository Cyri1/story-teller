<template>
  <ion-page>
    <ion-content :fullscreen="true">
      <div id="container">
        <swiper :loop="false" @init="storeSwiperInstance" @slideChangeTransitionEnd="readAudioActiveSlide">
          <swiper-slide v-for="(slide, index) in activeSlides" :key="index">
            <ion-img @click="storeActiveStoryIndex(index), clickOk(slide.actionNode)" :src="convertPath(slide.image)">
            </ion-img>
          </swiper-slide>
        </swiper>
        <button @click="click" color="primary">
          Parse all stories in packs
        </button>
        <br />
        <button @click="click2" color="primary">log jsonStories</button>
        <br />
        <button @click="createStoriesIndex" color="primary">
          create array for slider
        </button>
        <br />
        <button @click="stop" color="primary">
          stop sound
        </button>
      </div>
    </ion-content>
  </ion-page>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { IonContent, IonPage, IonImg } from "@ionic/vue";
import { Filesystem, Directory, Encoding } from "@capacitor/filesystem";
import { Capacitor } from "@capacitor/core";
import { Swiper, SwiperSlide } from "swiper/vue";
import { Howl } from "howler";
import "swiper/css";
import "@ionic/vue/css/ionic-swiper.css";

let jsonStories = ref([])
let activeSlides = ref([])
let homeButton = ref({})
let activeStoryIndex = ref(null)
let swipe = ref()
let activeAudioSlideSrc = ref([])
let readAudioSrc = ref([])

var readAudioHowl = new Howl({
    src: [readAudioSrc.value[0]],
    onend: function () {
      swipe.value.emit('slideChangeTransitionEnd')
    }
  })
  
var activeAudioSlideHowl = new Howl({
    src: [activeAudioSlideSrc.value[0]],
  })

onMounted(() => {

})

function click() {
  Filesystem.readdir({
    path: "",
    directory: Directory.Documents,
  })
    .then((ReaddirResult) => {
      if (!ReaddirResult.files.includes("packs")) {
        Filesystem.mkdir({
          path: "packs/",
          directory: Directory.Documents,
          recursive: true,
        }).catch((e) => {
          console.log("mkdir packs folder error: " + e);
        });
      }
    })
    .then(() => {
      Filesystem.readdir({
        path: "packs/",
        directory: Directory.Documents,
      })
        .then((ReaddirResult) => {
          for (let storyDirName of ReaddirResult.files) {
            Filesystem.readFile({
              path: "packs/" + storyDirName + "/story.json",
              directory: Directory.Documents,
              encoding: Encoding.UTF8,
            })
              .then((ReaddirFile) => {
                var jsonStory = JSON.parse(ReaddirFile.data);
                jsonStory["name"] = storyDirName;
                jsonStories.value.push(jsonStory);
              })
              .catch((e) => {
                console.log(
                  "readFile" + storyDirName + " story.json error: " + e
                );
              });
          }
        })
        .catch((e) => {
          console.log("readdir error: " + e);
        });
    })
    .catch((e) => {
      console.log("mkdir error: " + e);
    });
}
function click2() {
  console.log(jsonStories.value);
  console.log(activeSlides.value);
  console.log(homeButton.value);
}
function stop() {
}
function storeSwiperInstance(swiper) {
  swipe.value = swiper
}

function storeActiveStoryIndex(index) {
  activeStoryIndex.value = index;
}

function createStoriesIndex() {
  activeSlides.value = [];
  for (var story of jsonStories.value) {
    var storyName = story.name;
    for (var node of story.stageNodes) {
      if (node.squareOne) {
        var slide = {
          audio:
            "file:///storage/emulated/0/Documents/packs/" +
            storyName +
            "/assets/" +
            node.audio,
          image:
            "file:///storage/emulated/0/Documents/packs/" +
            storyName +
            "/assets/" +
            node.image,
          actionNode: node.okTransition.actionNode,
        };
        activeSlides.value.push(slide);
      }
    }
  }
}
function clickOk(actionNodeID) {
  console.log(jsonStories.value[activeStoryIndex.value].name);
  var storyName = jsonStories.value[activeStoryIndex.value].name
  console.log("action node searched : ");
  console.log(actionNodeID);
  var nextStagesNodes = getNextStageNodes(actionNodeID);
  // return [0: {audio: xxx.mp3, controlSettings: {wheel:..., autoplay: true/false}, homeTransition, image, okTransition: {actionNode: xxx-xxx-xxx, optionIndex: 0}, }]
  console.log("next stage : ");
  console.log(nextStagesNodes);
  if (nextStagesNodes[0].controlSettings.autoplay) {
    // if autoplay = true, it means that node is audio of slide set
    readAudio(
      "file:///storage/emulated/0/Documents/packs/" +
      storyName +
      "/assets/" +
      nextStagesNodes[0].audio
    )
    clickOk(nextStagesNodes[0].okTransition.actionNode);
  } else if (nextStagesNodes.length > 0) {
    activeSlides.value = [];
    for (var node of nextStagesNodes) {
      var slide = {
        audio:
          "file:///storage/emulated/0/Documents/packs/" +
          storyName +
          "/assets/" +
          node.audio,
        image:
          "file:///storage/emulated/0/Documents/packs/" +
          storyName +
          "/assets/" +
          node.image,
        actionNode: node.okTransition.actionNode,
      };
      homeButton.value = { homeTransition: node.homeTransition };
      activeSlides.value.push(slide);
    }
  }
  else {
    console.log('else');
  }
}
function getNextStageNodes(actionNodeID) {
  for (var actionNode of jsonStories.value[activeStoryIndex.value]
    .actionNodes) {
    if (actionNode.id === actionNodeID) {
      var nextStageIds = actionNode.options;
      var nextStagesNodes = createArrayOfNodesIds(nextStageIds)

    }
  }
  return nextStagesNodes;
}
function createArrayOfNodesIds(nextStageIds) {
  var stageNodeIds = [];
  for (var stageOption of nextStageIds) {
    for (var stageNode of jsonStories.value[activeStoryIndex.value]
      .stageNodes) {
      if (stageNode.uuid === stageOption) {
        stageNodeIds.push(stageNode);
      }
    }
  }
  return stageNodeIds;
  // return [0: "id-id-id-id-id", ...]
}

function convertPath(path) {
  return Capacitor.convertFileSrc(path);
}

function readAudioActiveSlide(swiper) {
  activeAudioSlideSrc.value.push(convertPath(activeSlides.value[swiper.activeIndex].audio))
  console.log(activeAudioSlideHowl);
  activeAudioSlideHowl.load();
  activeAudioSlideHowl.play();
}

function readAudio(audioFilePath) {
  console.log('clicked slide sound :');
  console.log(audioFilePath);
  console.log(readAudioHowl);
  readAudioSrc.value.push(convertPath(audioFilePath))
  readAudioHowl.load()
  readAudioHowl.play();
}
</script>

<style scoped>
#container {
  text-align: center;
  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

#container strong {
  font-size: 20px;
  line-height: 26px;
}

#container p {
  font-size: 16px;
  line-height: 22px;
  color: #8c8c8c;
  margin: 0;
}

#container a {
  text-decoration: none;
}
</style>