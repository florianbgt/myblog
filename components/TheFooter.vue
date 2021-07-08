<template>
    <b-container fluid class="bg-secondary text-left text-light py-5">
        <b-container class="mx-auto" style="max-width: 800px">
            <b-row class="mb-3">
                <b-col sm=5 class="mb-2">
                    <a href="mailto:florian.bigot321@gmail.com" class="text-light">
                        <fa :icon="['fas', 'envelope']" style="font-size: 30px"/>
                        <span>florian.bigot321@gmail.com</span>
                    </a>
                </b-col>
                <b-col sm=3 class="mb-2">
                    <a href="tel: +1 401-316-8293" class="text-light">
                        <fa :icon="['fas', 'phone']" style="font-size: 30px"/>
                        <span>+1 401-316-8293</span>
                    </a>
                </b-col>
                <b-col sm=2 class="mb-2">
                    <a href="https://linkedin.com/in/florianbgt" target="_blank" rel="noopener" class="text-light">
                        <fa :icon="['fab', 'linkedin']" style="font-size: 30px"/>
                        <span>Linkedin</span>
                    </a>  
                </b-col>
                <b-col sm=2 class="mb-2">
                    <a href="https://github.com/florianbgt" target="_blank" rel="noopener" class="text-light">
                        <fa :icon="['fab', 'github']" style="font-size: 30px"/>
                        <span>Github</span>
                    </a>
                </b-col>
            </b-row>
            <div class="text-sm-center text-left">
                &copy; {{new Date().getFullYear()}} <strong>Florian Bigot</strong>
            </div>
        </b-container>
    </b-container>
</template>