<template>
  <main class="Home select" v-if="isMenu">
    <LogoShowIn />
    <compact-lang-switch  class="absolute-top-left q-ma-sm"/>
    <!-- <theme-switch class="absolute-top-right q-ma-sm" /> -->
    <h2>Daily</h2>
    <p>{{ t("explanation") }}</p>
    <p>{{ t("shortcut" + ($q.platform.is.mobile ? "Mobile" : "")) }}</p>
    <form>
        <WikiLangSelect :dense="$q.screen.lt.sm" />
        <q-field outlined :dense="$q.screen.lt.sm" label="Date" stack-label>
            <template v-slot:prepend>
                <q-icon name="event" />
            </template>
            <template v-slot:control>
                <div class="self-center full-width no-outline" tabindex="0">{{formatDate}}</div>
            </template>
            <q-popup-proxy>
            <q-date
                v-model="formatDate"
                :options="date => date <= currentDate"
            />
            </q-popup-proxy>
        </q-field>
        <q-field outlined :dense="$q.screen.lt.sm" stack-label>
            <q-checkbox v-model="reverse" label="Reverse" />
        </q-field>
    </form>
    <q-btn push :label="t('start')" @click="start()">
        <mdi-check-bold/>
    </q-btn>
    <News/>
    <section>
      <div>
        <p>{{ t('contribution.discord') }}</p>
        <discord-btn/>
      </div>
      <div>
        <p>{{ t('contribution.github') }}</p>
        <github-btn/>
      </div>
      <div>
        <p>{{ t('contribution.kofi') }}</p>
        <kofi-btn/>
      </div>
      <div>
        <p>{{ t('contribution.nano') }}</p>
        <nano-btn/>
      </div>
    </section>
  </main>
  <main class="Home play" v-show="!isMenu">
    <div class="timeleft">{{ formatTime }}</div><q-btn flat class="target-page-btn" @click="targetPageBtnClick">{{ touchSlide?.state == OpenState.right ? t("closeTargetPage") : t("openTargetPage") }}</q-btn>
    <TouchSlide ref="touchSlide">
      <template #core>
        <wiki-page @wikiLink="onWikiLink" ref="wikiPage" :title="startPageF + ' → ' + targetPageF" />
      </template>
      <template #right >
        <wiki-page disable ref="wikiTargetPage" :title="t('targetPage')" />
      </template>
    </TouchSlide>
  </main>
</template>
<style lang="scss">
.Home.select {
    overflow-x: hidden;
  min-height: 100vh;
  display: flex;
  align-items: center;
  flex-direction: column;
  padding: 15px;
  gap: 15px;
  .reverse-check {
      justify-content: center;
      border: 1px solid rgba(0, 0, 0, 0.24);
      border-radius: 4px;
  }
  .logo-show-in {
    --logo-height: min(8em, 25vw);
  }
  section {
    width: 100%;
    display: flex;
    justify-content: space-evenly;
    flex-wrap: wrap;
    gap: 15px;
    div {
      display: grid;
      place-items: center;
    }
  }

  form {
    display: flex;
    width: 100%;
    flex-wrap: wrap;
    justify-content: center;
    gap: 15px;
    > * {
      min-width: 26ch;
    }
  }

}

.Home.play {
    min-height: 100%;
    .timeleft {
        z-index: 2;
        position: absolute;
        border-bottom-right-radius: 15px;
        backdrop-filter: blur(1.5px) brightness(1.2);
        border: 1px solid #0001;
        font-weight: bolder;
        top: 0;
        left: 0;
        margin: 0;
        padding: 0 4px 2px 5px;
    }

    .target-page-btn {
        z-index: 2;
        position: absolute;
        backdrop-filter: blur(1.5px) brightness(1.2);
        border: 1px solid #0001;
        border-radius: 0 0 20px 20px;
        top: 0;
        right: 25px;
        margin: 0;
    }

    .touch-slide {
        min-height: 100%;
        .left-slide {
            width: 0;
            height: 0;
        }
        .right-slide {
            width: 100%;
            min-height: 100%;
            background: #fff;
            box-shadow: 0 0 10px #0005;
            // filter: drop-shadow(0 0 5px #000);
        }
    }

}
</style>
<script lang="ts" setup>
import { computed, onUnmounted, ref, watch } from "vue";
import { useQuasar } from 'quasar';
import { useMagicKeys } from '@vueuse/core';
import WikiPage from "../components/WikiPage.vue";
import LogoShowIn from "../components/art/LogoShowIn.vue";
import WikiLangSelect from "src/components/form/WikiLangSelect.vue";
import { useI18n } from "vue-i18n";
import { wikiLang } from "src/components/form/wikiLang";
import { useRoute, useRouter } from "vue-router";
import { useTimer } from "src/composables/useTimer";
import TouchSlide, { OpenState } from "src/components/layouts/TouchSlide.vue";
import CompactLangSwitch from 'src/components/setting/CompactLangSwitch.vue';
import ThemeSwitch from 'src/components/setting/ThemeSwitch.vue';
import GithubBtn from "src/components/button/GithubBtn.vue";
import NanoBtn from "src/components/button/NanoBtn.vue";
import KofiBtn from "src/components/button/KofiBtn.vue";
import DiscordBtn from "src/components/button/DiscordBtn.vue";
import MdiCheckBold from '~icons/mdi/check-bold';
import News from "src/components/extra/News.vue";

const { t } = useI18n({ useScope: "local" });
const $q = useQuasar();
const router = useRouter();
const route = useRoute();

const reverse = ref(false);
const isMenu = ref(true);
const wikiPage = ref<InstanceType<typeof WikiPage>>();
const wikiTargetPage = ref<InstanceType<typeof WikiPage>>();
const touchSlide = ref<InstanceType<typeof TouchSlide>>();

const now = new Date();
const z = (n:number, p:number) => String(n).padStart(p, '0');
const currentDate = `${z(now.getUTCFullYear(),4)}/${z(now.getUTCMonth()+1,2)}/${z(now.getUTCDate(),2)}`;
const formatDate = ref(currentDate.slice());

const {
  timeController,
  timeStamp,
  time,
  startTimer
} = useTimer();

const formatTime = computed(()=>{
  const min = Math.floor(time.value/60)
  const minS = min > 0 ? min+"m" : "";
  const secS = time.value % 60+"s";
  return minS+secS;
})

const startPage = ref("");
const endPage = ref("");

const apiUrl = route.query.apiUrl as string;

const startPageF = computed(()=> decodeURI(startPage.value).replace(/_/g," "));
const targetPageF = computed(()=> decodeURI(endPage.value).replace(/_/g," "));

const history = ref<string[]>([]);

function onWikiLink(url: string) {
    history.value.push(url);
    if (url.replaceAll(/\s/g, "_") == decodeURI(endPage.value)) {
        onWin();
    }
}

function onWin() {
    router.push(`/result/${wikiLang.value}?time=${time.value}&path=${history.value.map(s=>encodeURIComponent(s)).join("|")}`)
}

function start() {
    // isMenu.value = false;
    // return;
    const url = apiUrl || `/daily/${wikiLang.value}/${formatDate.value}`;
    console.log("url : ", url);
    fetch(url, {
        redirect: "error"
    })
        .then(async r => {return { text: await r.text(), res: r}})
        .then(r => {
            const { res, text } = r;
            if (!res.ok || text.startsWith("<!DOCTYPE html>")) {
                return $q.notify({
                    type: 'negative',
                    position: 'top',
                    message: t('noDaily')
                });
            }
            [startPage.value, endPage.value] = text.split(/\r*\n/g);
            if (reverse.value) [startPage.value, endPage.value] = [endPage.value, startPage.value];
            isMenu.value = false;
            wikiPage.value!.requestWikiPage(decodeURI(startPage.value))
                .then(() => {
                    startTimer();
                });
            wikiTargetPage.value!.requestWikiPage(decodeURI(endPage.value));
        })
}



const keys = useMagicKeys()
const shiftCtrlA = keys['Ctrl+Alt+Space']

watch(shiftCtrlA, v => v && targetPageBtnClick());

function targetPageBtnClick() {
    switch (touchSlide.value?.state) {
        case OpenState.nothing: return touchSlide.value?.changeState(OpenState.right);
        case OpenState.right: return touchSlide.value?.changeState(OpenState.nothing);
    }
}

onUnmounted(() => {
    timeController.value.abort();
});
</script>
<i18n lang="yaml">
    en:
        start: "Start"
        noDaily: "No daily available for this language"
        apiError: "The custom daily provider doesn't repect the format"
        explanation: |
            Welcome to Wiki Adventure Daily! The goal is to go from 1 wikipedia page to an other by following link.
            Try to finish this daily challenge as fast you can with the fewest link possible and share you adventure with your friend!

        shortcut: "You can see the goal page with ctrl + alt + space ."
        shortcutMobile: "You can swipe to the left to see the goal page."
        targetPage: "Target page"
        openTargetPage: "Open target page"
        closeTargetPage: "Close target page"
        contribution:
            discord: "Find friends to play with on our Discord"
            github: "Contribute to the game"
            kofi: "Buy me a coffee"
            nano: "Share your Nano with me !"

    fr:
        start: "Commencer"
        noDaily: "Pas de daily disponible pour cette langue"
        apiError: "Le fournisseur de daily personnalisé ne respecte pas le format"
        explanation: |
            Bienvenue sur Wiki Adventure Daily! Le but est d'aller d'une page wikipédia à une autre en suivant uniquement les liens.
            Essayer de finir le challenge quotidien aussi vite que vous pouvez avec le moins de liens possible et partager votre aventure avec vos amis!

        shortcut: "Vous pouvez consulter la page cible avec ctrl + alt + space ."
        shortcutMobile: "Vous pouvez consulter la page cible en glissant vers la gauche."
        targetPage: "Page cible"
        openTargetPage: "Ouvrir la page cible"
        closeTargetPage: "Fermer la page cible"
        contribution:
            discord: "Trouve des joueurs sur notre Discord"
            github: "Apporte ta pierre à l'édifice grâce à Github"
            kofi: "Offre moi un café"
            nano: "Partage tes Nano avec moi"
</i18n>
