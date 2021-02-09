<template>
  <HeaderSidebarLayout v-if="this.repo != null">
    <template v-slot:header>
      <!-- TODO: Per-product styling! -->

      <!-- Header (Desktop) -->
      <div class="desktop-only">
        <div
          :class="[product.classes.bg, product.classes.text]"
          class="py-20 grid grid-cols-10 gap-4"
        >
          <div class="col-start-2 col-span-7">
            <h1 class="text-3xl font-semibold">
              {{ repo.metadata.name }}
            </h1>
            <p class="mt-2">
              {{ repo.metadata.longDescription }}
            </p>

            <a
              target="blank"
              :href="`https://github.com/${repo.metadata.owner}/${repo.metadata.repo}`"
            >
              <MaterialButton type="secondary" class="mt-8">
                View on GitHub
                <font-awesome-icon icon="external-link-alt" class="ml-1" />
              </MaterialButton>
            </a>
          </div>

          <div class="col-start-9 col-span-2e">
            <p class="pt-1">
              {{ repo.stats.stars }}
              <font-awesome-icon fixed-width icon="star" />
            </p>
            <p class="pt-1">
              {{ repo.stats.forks }}
              <font-awesome-icon fixed-width icon="code-branch" />
            </p>
          </div>
        </div>
      </div>

      <!-- Header (Mobile) -->
      <div class="mobile-only">
        <div
          :class="[product.classes.bg, product.classes.text]"
          class="py-4 px-8"
        >
          <h1 class="text-2xl font-semibold">
            {{ repo.metadata.name }}
          </h1>
          <p class="mt-1">
            {{ repo.metadata.shortDescription }}
          </p>
          <p class="mt-1 text-sm">
            <span class="mr-2">
              {{ repo.stats.stars }}
              <font-awesome-icon fixed-width size="sm" icon="star" />
            </span>
            <span>
              {{ repo.stats.forks }}
              <font-awesome-icon fixed-width size="sm" icon="code-branch" />
            </span>
          </p>
        </div>
      </div>
    </template>

    <template v-slot:sidebar>
      <!-- Side bar -->
      <div>
        <p class="uppercase font-medium mt-4 mb-2">{{ product.name }}</p>
        <ul class="text-sm">
          <li>
            <router-link :to="`/products/${productKey}`"
              >All Projects</router-link
            >
          </li>
          <li><a :href="product.docsUrl" target="_blank">Official Docs</a></li>
        </ul>

        <p class="uppercase font-medium mt-4 mb-2">Project</p>
        <ul class="text-sm">
          <li><router-link :to="fullPagePath()">Home</router-link></li>
          <li v-for="p in repo.metadata.pages" :key="p.path">
            <router-link :to="fullPagePath(p.path)">{{ p.name }}</router-link>
          </li>
        </ul>

        <div v-if="repo.metadata.links">
          <p class="uppercase font-medium mt-4 mb-2">Links</p>
          <ul class="text-sm">
            <li v-for="l in repo.metadata.links" :key="l.href">
              <a :href="l.href" target="_blank">{{ l.title }}</a>
            </li>
          </ul>
        </div>
      </div>
    </template>

    <!-- Body -->
    <div class="grid grid-cols-10 gap-4 mb-10 lg:mb-20">
      <div
        v-if="content != null"
        class="col-span-10 px-8 lg:px-0 lg:col-start-2 lg:col-span-8"
      >
        <template v-for="(s, i) in content.sections">
          <h2 v-if="i > 0" class="text-2xl mt-8 mb-2" :key="`header-${s.name}`">
            {{ s.name }}
          </h2>
          <!-- The 'prose' class comes from the Tailwind typography plugin -->
          <div
            class="prose mt-4 lg:mt-8"
            :key="`content-${s.name}`"
            v-html="s.content"
          ></div>
        </template>
      </div>
    </div>
  </HeaderSidebarLayout>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator";
import { getModule } from "vuex-module-decorators";

import MaterialButton from "@/components/MaterialButton.vue";
import HeaderSidebarLayout from "@/components/HeaderSidebarLayout.vue";
import { ProductConfig, ALL_PRODUCTS } from "@/model/product";
import ProjectsModule from "@/store/project";
import UIModule from "@/store/ui";
import { firestore } from "@/plugins/firebase";

import { RepoData, RepoPage } from "../../../shared/types";
import * as util from "../../../shared/util";

@Component({
  components: {
    MaterialButton,
    HeaderSidebarLayout,
  },
})
export default class Repo extends Vue {
  public product!: ProductConfig;
  public repo: RepoData | null = null;
  public content: RepoPage | null = null;

  private productKey!: string;
  private id!: string;

  private projectsModule = getModule(ProjectsModule, this.$store);
  private uiModule = getModule(UIModule, this.$store);

  async mounted() {
    this.productKey = this.$route.params["product"];
    this.id = this.$route.params["repo"];
    this.product = ALL_PRODUCTS[this.productKey];

    const p = this.loadContent();
    this.uiModule.waitFor(p);
  }

  async loadContent() {
    const opts = { product: this.productKey, id: this.id };
    await this.projectsModule.fetchRepo(opts);
    this.repo = this.projectsModule.repoByProductAndId(opts);

    if (this.repo === null) {
      throw new Error("Could not load repo!");
    }

    const page =
      this.$route.params["page"] ||
      util.cleanPagePath(this.repo.metadata.content);

    // TODO: Where should this be done?
    const pageKey = btoa(page);
    const pageRef = firestore()
      .collection("products")
      .doc(this.productKey)
      .collection("repos")
      .doc(this.repo.id)
      .collection("pages")
      .doc(pageKey);

    const snap = await pageRef.get();
    this.content = snap.data() as RepoPage;
  }

  public fullPagePath(path?: string) {
    const base = `/products/${this.productKey}/repos/${this.repo?.id}`;
    if (!path) {
      return base;
    }

    return `${base}/pages/${util.cleanPagePath(path)}`;
  }
}
</script>

<style scoped lang="postcss">
.router-link-exact-active {
  @apply font-bold text-blue-500;
}

a {
  @apply hover:underline;
}
</style>