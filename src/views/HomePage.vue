<template>
  <ion-page>
    <ion-content :fullscreen="true">
      <div id="container">
        <swiper :loop="false" @slideChangeTransitionEnd="readAudio">
          <swiper-slide v-for="(slide, index) in activeSlides" :key="index">
            <ion-img @click="storeActiveStoryIndex(index), okTransition(slide.okTransition.actionNode, slide.okTransition.optionIndex)"
              :src="convertPath(slide.image)"
            ></ion-img>
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
      </div>
    </ion-content>
  </ion-page>
</template>

<script>
import { IonContent, IonPage, IonImg } from "@ionic/vue";
import { Filesystem, Directory, Encoding } from "@capacitor/filesystem";
import { Capacitor } from "@capacitor/core";
import { Swiper, SwiperSlide } from "swiper/vue";
import "swiper/css";
import "@ionic/vue/css/ionic-swiper.css";
export default {
  name: "HomePage",
  components: {
    Swiper,
    SwiperSlide,
    IonContent,
    IonImg,
    IonPage,
  },
  data() {
    return {
      jsonStories: [],
      activeSlides: [],
      homeButton: {},
      activeStoryIndex: null,
    };
  },
  mounted() {},
  methods: {
    click() {
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
                    this.jsonStories.push(jsonStory);
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
    },
    click2() {
      console.log(this.jsonStories);
      console.log(this.activeSlides);
    },
    storeActiveStoryIndex(index) {
      this.activeStoryIndex = index;
    },
    createStoriesIndex() {
      for (var story of this.jsonStories) {
        var storyName = story.name;
        for (var node of story.stageNodes) {
          if (node.squareOne) {
            var obj = {
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
              okTransition: {
                actionNode: node.okTransition.actionNode,
                optionIndex: node.okTransition.optionIndex,
              },
            };
            this.activeSlides.push(obj);
            this.homeButton = { homeTransition: node.homeTransition };
          }
        }
      }
    },
    okTransition(actionNodeID, optionIndex) {
      for (var actionNode of this.jsonStories[this.activeStoryIndex].actionNodes) {
        if(actionNode.id == actionNodeID) {
          console.log(actionNode.options[optionIndex]);
        }
      }
    },
    convertPath(path) {
      return Capacitor.convertFileSrc(path);
    },
    readAudio(swiper) {
      var audioFilePath = this.convertPath(this.activeSlides[swiper.activeIndex].audio);
      var playAudio = new Audio(audioFilePath);
      playAudio.play();
    },
  },
};
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