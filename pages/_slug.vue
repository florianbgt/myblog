<template>
    <b-container fluid>
        <b-container class="py-5">
            <blog-img v-if="'image' in article" :src="article.image" alt="Head image" style="width: 100%; height: 250px; object-fit: cover"/>
            <img v-else :src="require('~/content/default.png')" style="width: 100%; height: 250px; object-fit: cover"/>
            <h1 class="mt-4">{{article.title}}</h1>
            <h4 class="text-secondary">
                <small>
                    <i>
                        <fa :icon="['fas', 'user']" style="font-size: 20px"/>
                        {{article.writtenBy}}
                        -
                        <fa :icon="['far', 'calendar-alt']" style="font-size: 20px"/>
                        {{ new Date(article.createdAt).toLocaleDateString('en', { year: 'numeric', month: 'long', day: 'numeric' }) }}
                    </i>
                </small>
            </h4>
            <nuxt-content :document="article" />
        </b-container>
    </b-container>
</template>

<script>
export default {
    async asyncData({ $content, params }) {
        const article = await $content(params.slug, 'index').fetch();
        return { article }
    }
}
</script>