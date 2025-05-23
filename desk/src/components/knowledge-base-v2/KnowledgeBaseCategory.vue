<template>
  <div
    class="flex flex-col p-5 px-10 w-full overflow-hidden"
    v-if="
      !categoryTreeResource.isLoading &&
      (!!category.subCategories.length || !!category.articles.length)
    "
  >
    <!-- Top Section -->
    <section class="flex flex-col gap-3.5 mb-5">
      <h3 class="text-2xl font-semibold text-gray-800">
        {{ category.categoryName || "Getting Started" }}
      </h3>
      <FormControl
        type="text"
        class="w-full"
        placeholder="Search (title, subtitle, author)"
        size="md"
        v-model="categorySearch"
        @input="searchArticles"
      >
        <template #prefix>
          <Icon icon="lucide:search" class="h-4 w-4 text-gray-500" />
        </template>
      </FormControl>
    </section>
    <div class="overflow-scroll">
      <!-- Sub categories Section -->
      <section
        class="flex flex-col gap-3 mb-8"
        v-if="!!category.subCategories.length"
      >
        <h3 class="text-lg font-semibold text-gray-900">Sub-categories</h3>
        <!-- sub category card container-->
        <div class="flex gap-5 flex-wrap text-lg">
          <!-- sub category card -->
          <div
            v-for="subCategory in category.subCategories"
            class="border rounded px-3.5 py-3 cursor-pointer max-w-[190px] min-w-[190px] hover:border-gray-500"
            @click="handleSubCategoryClick(subCategory)"
          >
            <h5 class="truncate text-lg">{{ subCategory?.category_name }}</h5>
            <span class="text-sm text-gray-600">
              {{ subCategory.articles.length }} articles
            </span>
          </div>
        </div>
      </section>
      <!-- Article List View -->
      <section class="flex flex-col gap-3" v-if="!!_articles.length">
        <h4 class="text-lg font-semibold text-gray-900">
          {{ showAllArticles ? "All Articles" : "Articles" }}
        </h4>
        <!-- Article Container -->
        <div class="flex flex-col gap-x-2 divide-y max-w-full">
          <!-- Article Card -->
          <ArticleCard
            v-for="article in _articles"
            :article="article"
            :author="category.authors[article.author]"
            :key="article.name"
            class="hover:bg-gray-200 transition-all hover:rounded"
          />
        </div>
      </section>
      <div
        v-else
        class="flex items center justify-center h-[300px] w-full text-gray-600"
      >
        No articles found
      </div>
    </div>
  </div>
  <div
    v-else
    class="flex items-center justify-center h-[300px] w-full text-gray-600"
  >
    No articles found
  </div>
</template>

<script setup lang="ts">
import { reactive, watch, ref, Reactive } from "vue";
import { createResource, FormControl, createListResource } from "frappe-ui";
import ArticleCard from "@/components/knowledge-base-v2/ArticleCard.vue";
import { Article, Author, Category, SubCategory } from "@/types";
import { Icon } from "@iconify/vue";
import { useRouter } from "vue-router";
import { useUserStore } from "@/stores/user";

interface P {
  categoryId?: string;
  showAllArticles?: boolean;
}
const props = withDefaults(defineProps<P>(), {
  categoryId: "",
  showAllArticles: false,
});

const router = useRouter();
const userStore = useUserStore();

const category: Reactive<Category> = reactive({
  categoryName: "",
  subCategories: [],
  articles: [],
  authors: {},
});

const categorySearch = ref("");
const _articles = ref([]);

const categoryTreeResource = createResource({
  url: "helpdesk.api.kbase.get_sub_categories_and_articles",
  name: props.categoryId,
  cache: ["category", props.categoryId],
  params: {
    category: props.categoryId,
  },
  auto: !props.showAllArticles,
});

const allArticles = createListResource({
  doctype: "HD Article",
  fields: [
    "name",
    "title",
    "category",
    "published_on",
    "author",
    "subtitle",
    "article_image",
    "_user_tags",
  ],
  filters: {
    status: "Published",
  },
  pageLength: 100,
  auto: props.showAllArticles,
  onSuccess(articles: Article[]) {
    category.articles = articles;
    _articles.value = articles;
    const authors = [...new Set(articles.map((article) => article.author))];
    category.authors = authors.reduce((acc, author) => {
      const authorInfo = userStore.getUser(author);
      acc[author] = {
        name: authorInfo.full_name ?? authorInfo.email,
        image: authorInfo.user_image ?? "",
      };
      return acc;
    }, {});
  },
});

function searchArticles() {
  const search = categorySearch.value.toLowerCase();
  const articles = category.articles.filter(
    (article) =>
      article.title.toLowerCase().includes(search) ||
      category.authors[article.author].name.toLowerCase().includes(search)
  );
  _articles.value = articles;
}

function handleSubCategoryClick(subCategory: SubCategory) {
  categoryTreeResource.update({
    params: {
      category: subCategory.name,
    },
  });
  categoryTreeResource.reload();
  router.push({
    query: {
      category: categoryTreeResource.data.root_category.category_id,
      subCategory: subCategory.name,
    },
  });
}

watch(
  () => router.currentRoute.value.query,
  () => {
    const { category, subCategory } = router.currentRoute.value.query as {
      category: string;
      subCategory: string;
    };
    if (!subCategory && category !== "Explore all articles") {
      categoryTreeResource.update({
        params: {
          category: category,
        },
      });
      categoryTreeResource.reload();
    }
  }
);

watch(
  async () => categoryTreeResource.data,
  async () => {
    const data = await categoryTreeResource.data;
    category.categoryName = data["root_category"].category_name;
    category.subCategories = data["sub_categories"];
    category.articles = data["all_articles"];
    category.authors = data["authors"];
    _articles.value = category.articles;
    return data;
  }
);

watch(
  () => props.categoryId,
  () => {
    categoryTreeResource.update({
      params: {
        category: props.categoryId,
      },
    });
    categoryTreeResource.reload();
  }
);
</script>

<style scoped></style>
