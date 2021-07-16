<template>
    <b-container fluid>
        <b-container class="py-5">
            <h1>Welcome to my <span class="text-secondary">Tech Blog!</span></h1>
            <div>
                <p>I am writting tutorials here on my free time about the techs I like to use</p>
                <p>Feel free to email me if you wanna chat about my content &#128512;</p>
                <p>
                    You can find all the content off this blog (and more!) on my github:
                    <a href="https://github.com/florianbgt" target="_blank" rel="noopener">https://github.com/florianbgt</a>
                </p>
                <p>
                    You can also find out more about me here:
                    <a href="https://florianbgt.com" target="blank">https://florianbgt.com</a>
                </p>
            </div>
            <b-row>
                <b-col>
                    <b-form-group label="Search by keywords">
                        <b-form-tags v-model="search" remove-on-delete separator=" "/>
                    </b-form-group>
                </b-col>
            </b-row>
            <b-row>
                <b-col v-for="(article, index) in articles" :key="index" lg=4 md=6 class="mb-2">
                    <b-card class="h-100">
                        <img :src="'image' in article ? require(`~/content${article.path}/${article.image}`) : require('~/content/default.png')" alt="Article image" style="width: 100%; height: 150px; object-fit: cover"/>
                        <h2>{{ article.title }}</h2>
                        <b-card-text>{{ article.description }}</b-card-text>
                        <div class="text-center">
                            <b-button :to="article.path" variant="secondary">Read</b-button>
                        </div>
                    </b-card>
                </b-col>
            </b-row>
        </b-container>
    </b-container>
</template>

<script>
export default {
    data () {
        return {
            search: []
        }
    },

    async asyncData({ $content, params }) {
        const fetchedArticles = await $content({deep:true}).fetch();
        fetchedArticles.sort((a, b) => new Date(b.updatedAt) - new Date(a.updatedAt));
        fetchedArticles.forEach(article => {
            article.path = article.path.split('/');
            article.path.splice(article.path.length-1, 1);
            article.path = article.path.join('/');
        });
        return { fetchedArticles };
    },

    computed: {
        articles: function () {
            if (this.search.length <= 0) {return this.fetchedArticles}
            else {
                return this.fetchedArticles.filter(article => {
                    return this.search.some((el) => {
                        console.log(el);
                        console.log(article.title.replace(/_/g, ' ').toLowerCase());
                        return article.title.replace(/_/g, ' ').toLowerCase().includes(el.toLowerCase());
                    })
                });
            }
        },
    }
}
</script>