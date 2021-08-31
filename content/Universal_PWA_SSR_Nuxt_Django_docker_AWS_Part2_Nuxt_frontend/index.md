---
title: Production ready Universal PWA with SSR using Nuxt, Django, Docker and AWS (Part 2)
description: Create an universal PWA with SSR and deploy it on AWS usign Nuxt, Django and Docker. Part 2, Nuxt frontend
writtenBy: "Florian Bigot"
---

In this article, we will setup a production ready universal PWA using Nuxt, Django, Docker and AWS.

This is the part 2 of the tutorial where we will setup the universal PWA using Nuxt.

Part 1 here: https://blog.florianbgt.com/http://localhost:3000/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part1_Django_API

Part 3 here: https://blog.florianbgt.com/http://localhost:3000/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part3_AWS_Deployment

You can find the source code on my github: https://github.com/florianbgt/Universal_PWA_SSR_Nuxt_Django_Docker_AWS

## 1) Setting up our project

To setup our project, we use the following command:

```bash
npm create nuxt-app pwa
```

<div><blog-img src="nuxt_create.PNG" alt="Create nuxt app" width="100%" height="auto" class="shadow mb-3"/></div>

We then delete the `node_modules` directory. We will not need it as we are going to dockerize the app.

```bash
rm -r pwa/node_modules
```

we now dockerize our app.

We start by creating the `Dockerfile`.

```bash
touch pwa/Dockerfile
```

```yaml
### pwa/Dockerfile
FROM node:16.6
WORKDIR /code
COPY package*.json /code/
RUN npm install
COPY . .
```

We then update our `docker-compose.yml` to include our new service.

```yaml
### docker-compose.yml
version: "3.9"

services:
  db:
    image: postgres
    volumes:
      - ./db:/var/lib/postgresql/data
    env_file:
      - .env
  api:
    restart: always
    build:
      context: api
      dockerfile: Dockerfile
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - ./api:/code
      - ./media:/media
    ports:
      - "8000:8000"
    depends_on:
      - db
    env_file:
      - .env
  pwa:
    restart: always
    build:
      context: pwa
      dockerfile: Dockerfile
    command: npm run dev
    volumes:
      - ./pwa:/code
      - /code/node_modules
      - /code/.nuxt
    ports:
      - "3000:3000"
    env_file:
      - .env
```

```txt
### .env
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_HOST=db
POSTGRES_PORT=5432

EMAIL_HOST=smtp.gmail.com
EMAIL_HOST_USER=florian.bigot321@gmail.com
EMAIL_HOST_PASSWORD=jlycddpgxhquttxc
EMAIL_PORT=587
DEFAULT_FROM_EMAIL=florian.bigot321@gmail.com

HOST=0.0.0.0
```

We can now spin up our containers.

```
docker-compose up
```

Here http://localhost:3000, we now see our nuxt app up and running

<div><blog-img src="nuxt_landing.PNG" alt="Nuxt landing page" width="100%" height="auto" class="shadow mb-3"/></div>

**On windows**, to enable hot reloading you need to had the following to `nuxt.config.js`:

```javascript
### pwa/nuxt.config.js
...
watchers: {
  webpack: {
    poll: true
  }
},
...
```

## 2) Create layouts and some pages

Let's create the layouts of our app, as well as the home and login page.

We will use `bootstrap-vue` for styling.

First, let's deleting Nuxt default page and components.

```bash
rm pwa/components/NuxtLogo.vue
rm pwa/components/Tutorial.vue
rm pwa/pages/index.vue
```

Now, let's create the files below.

```bash
mkdir pwa/layouts
touch pwa/components/Header.vue
touch pwa/components/Footer.vue
touch pwa/layouts/default.vue
touch pwa/layouts/headless.vue
touch pwa/layouts/error.vue
touch pwa/page/index.vue
touch pwa/page/login.vue
```

```vue
### pwa/components/header.vue
<template>
  <div>
    <div style="height: 65px" />
    <b-navbar toggleable="md" fixed="top" type="dark" variant="dark">
      <b-container>
        <b-navbar-brand to="/">
          <img
            src="/icon.png"
            height="40px"
            width="50px"
            alt="logo"
            class="d-inline-block bg-light p-1 rounded"
          />
          <span class="ml-2"> Awesome Recipes! </span>
        </b-navbar-brand>

        <b-navbar-toggle target="nav-collapse"></b-navbar-toggle>

        <b-collapse id="nav-collapse" is-nav>
          <b-navbar-nav>
            <b-nav-item to="/"> Home </b-nav-item>
            <b-nav-item to="/recipes"> Recipes </b-nav-item>
            <b-nav-item to="/profile"> Profile </b-nav-item>
            <b-nav-item @click="logout"> Logout </b-nav-item>
          </b-navbar-nav>
        </b-collapse>
      </b-container>
    </b-navbar>
  </div>
</template>

<script>
export default {
  methods: {
    async logout() {
      try {
        console.log("logout");
      } catch (err) {
        console.log(err);
      }
    }
  }
};
</script>
```

```vue
### pwa/components/Footer.vue
<template>
  <b-container
    fluid
    class="text-center bg-dark text-light border-top border-2 p-3"
  >
    <div>
      <strong>
        <a href="mailto: florian.bigot@hutchinsonna.com" class="text-light">
          florian.bigot@hutchinsonna.com
        </a>
      </strong>
    </div>
  </b-container>
</template>
```

```vue
### pwa/layouts/default.vue
<template>
  <div class="d-flex flex-column text-dark" style="min-height: 100vh">
    <Header />
    <b-container fluid class="bg-light py-4" style="flex: 1">
      <b-container class="px-0">
        <Nuxt />
      </b-container>
    </b-container>
    <Footer />
  </div>
</template>
```

```vue
### pwa/layouts/headless.vue
<template>
  <div
    class="text-dark d-flex justify-content-center align-items-center"
    style="width: 100vw; height: 100vh"
  >
    <Nuxt />
  </div>
</template>
```

```vue
### pwa/layouts/error.vue
<template>
  <div>
    <h1 v-if="error.statusCode === 404">
      Page Not Found ({{ error.statusCode }})
    </h1>
    <h1 v-else-if="error.statusCode === 500">
      An error occured on the server ({{ error.statusCode }})
    </h1>
    <h1 v-else>An error occured ({{ error.statusCode }})</h1>
    <div>{{ error.message }}</div>
    <NuxtLink to="/">Go back to the Home page</NuxtLink>
  </div>
</template>

<script>
export default {
  props: ["error"],
  layout: "headless"
};
</script>
```

```vue
### pwa/page/index.vue
<template>
  <div>
    <p><strong>Welcome to awesome recipes!</strong></p>
    <p>Start browsing awesome <NuxtLink to="/recipes">recipes</NuxtLink></p>
  </div>
</template>
```

```vue
### pwa/page/login.vue
<template>
  <div class="w-100" style="max-width: 500px">
    <b-card title="Log In" class="bg-secondary text-light">
      <b-form @submit.prevent="login">
        <b-form-group id="email" label="Email" label-for="input-email">
          <b-form-input
            v-model="email"
            id="input-email"
            type="email"
            placeholder="Enter Email"
            autocomplete="username"
            required
            :disabled="loading"
          />
        </b-form-group>
        <b-form-group
          id="password"
          label="Password"
          label-for="input-password"
        >
          <b-form-input
            v-model="password"
            id="input-password"
            type="password"
            placeholder="Enter Password"
            autocomplete="current-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <div class="text-center">
          <b-button type="submit" variant="light">Log In</b-button>
        </div>
        <div class="text-center mt-2">
          <b-link to="/password/reset" class="text-light"
            >Forgot your password</b-link
          >
          |
          <b-link to="/signup" class="text-light">Sign Up</b-link>
        </div>
      </b-form>
    </b-card>
  </div>
</template>

<script>
export default {
  layout: "headless",
  data() {
    return {
      email: "",
      password: "",
      loading: false
    };
  },
  methods: {
    async login() {
      console.log("login");
    }
  }
};
</script>
```

Our `home` and `login` page are now created! They should look like below:

<div><blog-img src="home.PNG" alt="home page" width="100%" height="auto" class="shadow mb-3"/></div>

<div><blog-img src="login.PNG" alt="login page" width="100%" height="auto" class="shadow mb-3"/></div>

## 3) Authentication

To manage authentication logic, we are going to use the `nuxt/auth` library.

To install it, run the following command.

```bash
docker-compose run pwa npm install --exact @nuxtjs/auth-next
```

To set up `nuxt/auth`, all we need to do is to add the following in our `nuxtconfig.js`:

```javascript
### pwa/nuxt.config.js
...
...
modules: [
  'bootstrap-vue/nuxt',
  '@nuxtjs/axios',
  '@nuxtjs/auth-next',
  '@nuxtjs/pwa',
],

router: {
  middleware: ["auth"],
},

axios: {
  progress: true,
},

publicRuntimeConfig: {
    axios: {
      browserBaseURL: process.env.API_URL,
    },
  },

  privateRuntimeConfig: {
    axios: {
      baseURL: "http://api:8000/",
    },
  },

auth: {
  strategies: {
    local: {
      scheme: 'refresh',
      token: {
        property: 'access',
      },
      refreshToken: {
        property: 'refresh',
        data: 'refresh',
      },
      user: {
        property: false,
      },
      endpoints: {
        login: { url: '/token/', method: 'post' },
        refresh: { url: '/token/refresh/', method: 'post' },
        user: { url: '/user/', method: 'get' },
        logout: false
      },
    }
  },
  redirect: {
    login: '/login',
    logout: '/login',
    home: '/'
  },
},
...
```

We also need to add the following to our `.env`:

```txt
### .env
POSTGRES_DB=postgres
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_HOST=db
POSTGRES_PORT=5432

EMAIL_HOST=smtp.gmail.com
EMAIL_HOST_USER=florian.bigot321@gmail.com
EMAIL_HOST_PASSWORD=jlycddpgxhquttxc
EMAIL_PORT=587
DEFAULT_FROM_EMAIL=florian.bigot321@gmail.com
FRONTEND_URL=http://localhost:3000

HOST=0.0.0.0
API_URL=http://localhost:8000/
```

We now can update our `login.vue` page to trigger login using `nuxt/auth`.

```javascript
### pwa/pages/login.vue
...
methods: {
  async login() {
    try {
      this.loading = true;
      await this.$auth.loginWith("local", {
        data: { email: this.email, password: this.password },
      });
    } catch (err) {
      console.log(err.response);
      this.$bvToast.toast(err.response.data.detail, {
        title: "Error while login",
        autoHideDelay: 5000,
        toaster: 'b-toaster-top-center',
        variant: "danger",
      });
    } finally {
      this.loading = false;
    }
  },
},
...
```

We also update our `Header.vue` component to trigger logout.

```javascript
### pwa/components/Header.vue
methods: {
  async logout() {
    try {
      this.$auth.logout()
    } catch (err) {
      console.log(err);
    }
  },
},
```

`nuxt/auth` uses Vuex in the background.

Nuxt automatically enable the store if the `store` directory contains files.

Thus, let's create an empty `recipes.js` file in ths store directory to enable Vuex store.

For now, this file remain empty.

```bash
touch pwa/store/recipe.js
```

We now can spin up our docker containerw and try to sign in/out users

```bash
docker-compose up --build
```

## 4) Signup

Next, let's create our Signup view!

For this, we create the `signup.vue` in the pages directory.

```bash
touch pwa/pages/signup.vue
```

```vue
### pwa/pages/signup.vue
<template>
  <div class="w-100" style="max-width: 500px">
    <b-card title="Sign Up" class="bg-secondary text-light">
      <b-form @submit.prevent="signup">
        <b-form-group id="email" label="Email" label-for="input-email">
          <b-form-input
            v-model="email"
            id="input-email"
            type="email"
            placeholder="Enter Email"
            autocomplete="username"
            required
            :disabled="loading"
          />
        </b-form-group>
        <b-form-group
          id="password"
          label="Password"
          label-for="input-password"
        >
          <b-form-input
            v-model="password"
            id="input-password"
            type="password"
            placeholder="Enter Password"
            autocomplete="new-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <b-form-group
          id="password2"
          label="Password"
          label-for="input-password"
        >
          <b-form-input
            v-model="password2"
            id="input-password2"
            type="password"
            placeholder="Repeat Password"
            autocomplete="new-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <div class="text-center">
          <b-button type="submit" variant="light">Sign Up</b-button>
        </div>
        <div class="text-center mt-2">
          <b-link to="/login" class="text-light">Already have an account? Log In instead</b-link>
        </div>
      </b-form>
    </b-card>
  </div>
</template>

<script>
export default {
  auth: "guest",
  layout: "headless",
  data() {
    return {
      email: "",
      password: "",
      password2: "",
      loading: false,
    };
  },
  methods: {
    async signup() {
      try {
        this.loading = true;
        this.$axios.setHeader("Authorization", null);
        await this.$axios.$post("signup/", {
          email: this.email,
          password: this.password,
          password2: this.password2,
        });
        await this.$auth.loginWith("local", {
          data: { email: this.email, password: this.password },
        });
      } catch (err) {
        console.log(err.response);
        this.$bvToast.toast(
          err.response.data.email ||
            err.response.data.password ||
            err.response.data.password2,
          {
            title: "Error while signup",
            autoHideDelay: 5000,
            toaster: "b-toaster-top-center",
            variant: "danger",
          }
        );
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

That is it, we can now Signup user.

Note that, user will be automatically signed in after sign up.

## 5) Email and password update

Let's create `profile.vue` in the page directory so users can change their email and password.

```bash
touch pwa/pages/profile.vue
```

```vue
### pwa/pages/profile.vue
<template>
  <div>
    <b-form @submit.prevent="changeEmail">
      <b-form-group id="email" label="Email" label-for="input-email">
        <b-form-input
          v-model="email.value"
          id="input-email"
          type="email"
          placeholder="Enter Email"
          autocomplete="username"
          required
          :disabled="loading || !email.edit"
        />
      </b-form-group>
      <template v-if="email.edit">
        <b-button @click="cancelEmail" variant="danger">Cancel</b-button>
        <b-button type="submit" variant="success">Validate</b-button>
      </template>
    </b-form>
    <template v-if="!email.edit">
      <b-button @click="email.edit = true">Change email</b-button>
      <b-button @click="password.edit = true">Change password</b-button>
    </template>
    <b-modal v-model="password.edit" title="Change password" hide-footer>
      <b-form @submit.prevent="changePassword">
        <b-form-group
          id="oldPassword"
          label="Old password"
          label-for="input-old-password"
        >
          <b-form-input
            v-model="password.old"
            id="input-old-password"
            type="password"
            placeholder="Enter old password"
            autocomplete="current-password"
            required
            :disabled="loading || !password.edit"
          />
        </b-form-group>
        <b-form-group
          id="newPassword"
          label="New password"
          label-for="input-new-password"
        >
          <b-form-input
            v-model="password.new"
            id="input-new-password"
            type="password"
            placeholder="Enter new password"
            autocomplete="new-password"
            required
            :disabled="loading || !password.edit"
          />
        </b-form-group>
        <b-form-group
          id="newPassword2"
          label="New password (again)"
          label-for="input-new-password2"
        >
          <b-form-input
            v-model="password.new2"
            id="input-new-password2"
            type="password"
            placeholder="Enter new password"
            autocomplete="new-password"
            required
            :disabled="loading || !password.edit"
          />
        </b-form-group>
        <b-button @click="cancelPassword" variant="danger">Cancel</b-button>
        <b-button type="submit" variant="success">Validate</b-button>
      </b-form>
    </b-modal>
  </div>
</template>

<script>
export default {
  async fetch() {
    this.email.value = this.$auth.user.email;
  },
  data: () => {
    return {
      email: { value: null, edit: false },
      password: { old: null, new: null, new2: null, edit: false },
      loading: false,
    };
  },
  methods: {
    cancelEmail() {
      this.email = { value: this.$auth.user.email, edit: false };
    },
    async changeEmail() {
      try {
        this.loading = true;
        const user = await this.$axios.$put("user/", {
          email: this.email.value,
        });
        await this.$auth.setUser(user);
        this.cancelEmail();
      } catch (err) {
        console.log(err);
        this.$bvToast.toast(err.response.data.email, {
          title: "Error while email change",
          autoHideDelay: 5000,
          toaster: "b-toaster-top-center",
          variant: "danger",
        });
      } finally {
        this.loading = false;
      }
    },
    cancelPassword() {
      this.password = { old: null, new: null, new2: null, edit: false };
    },
    async changePassword() {
      try {
        this.loading = true;
        await this.$axios.$put("password/change/", {
          old_password: this.password.old,
          password: this.password.new,
          password2: this.password.new2,
        });
        this.cancelPassword();
      } catch (err) {
        console.log(err);
        this.$bvToast.toast(
          err.response.data.old_password
            ? err.response.data.old_password.old_password
            : undefined ||
                err.response.data.password ||
                err.response.data.password2,
          {
            title: "Error while password change",
            autoHideDelay: 5000,
            toaster: "b-toaster-top-center",
            variant: "danger",
          }
        );
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

Now user can update their email and change their password.

## 6) Password reset

For password reset, we create the following files in the pages directory.

```bash
touch pwa/pages/password/reset/index.vue
touch pwa/pages/password/reset/confirm.vue
```

```vue
### pwa/pages/password/reset/index.vue
<template>
  <div class="w-100" style="max-width: 500px">
    <b-card title="Password reset" class="bg-secondary text-light">
      <b-form v-if="!sent" @submit.prevent="resetPassword">
        <b-form-group id="email" label="Email" label-for="input-email">
          <b-form-input
            v-model="email"
            id="input-email"
            type="email"
            placeholder="Enter Email"
            autocomplete="username"
            required
            :disabled="loading"
          />
        </b-form-group>
        <div class="text-center">
          <b-button to="/login" variant="light">Back to Login</b-button>
          <b-button type="submit" variant="success">Send reset link</b-button>
        </div>
      </b-form>
      <template v-else>
        <p>An Password reset link have been sent to <strong>{{ email }}</strong></p>
        <p>
          Did not receive the link?
          <b-button @click="sent = false" size="sm" variant="light"
            >Send a new one</b-button
          >
        </p>
      </template>
    </b-card>
  </div>
</template>

<script>
export default {
  auth: "guest",
  layout: "headless",
  data() {
    return {
      email: "",
      sent: false,
      loading: false,
    };
  },
  methods: {
    async resetPassword() {
      try {
        this.loading = true;
        await this.$axios.$post("password/reset/", { email: this.email });
        this.sent = true;
      } catch (err) {
        console.log(err.response);
        this.$bvToast.toast(err.response.data.email, {
          title: "Error while requesting password reset",
          autoHideDelay: 5000,
          toaster: "b-toaster-top-center",
          variant: "danger",
        });
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

```vue
### pwa/pages/password/reset/confirm.vue
<template>
  <div class="w-100" style="max-width: 500px">
    <b-card title="Password reset" class="bg-secondary text-light">
      <b-form v-if="tokenIsValid" @submit.prevent="resetPassword">
        <b-form-group
          id="password"
          label="Password"
          label-for="input-password"
        >
          <b-form-input
            v-model="password"
            id="input-password"
            type="password"
            placeholder="Enter Password"
            autocomplete="new-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <b-form-group
          id="password2"
          label="Password"
          label-for="input-password2"
        >
          <b-form-input
            v-model="password2"
            id="input-password2"
            type="password"
            placeholder="Enter Password"
            autocomplete="new-password"
            required
            :disabled="loading"
          />
        </b-form-group>
        <div class="text-center">
          <b-button type="submit" variant="success">Reset password</b-button>
        </div>
      </b-form>
      <template v-else>
        <p>
          You need a valid link to reset your password
          <b-button to="/password/reset" size="sm" variant="light"
            >Request one</b-button
          >
        </p>
      </template>
    </b-card>
  </div>
</template>

<script>
export default {
  auth: "guest",
  layout: "headless",
  async fetch() {
    await this.validateToken();
  },
  data() {
    return {
      password: null,
      password2: null,
      tokenIsValid: null,
      loading: false,
    };
  },
  computed: {
    token() {
      return this.$route.query.token;
    },
  },
  methods: {
    async validateToken() {
      try {
        await this.$axios.$post("password/reset/validate_token/", {
          token: this.token,
        });
        this.tokenIsValid = true;
      } catch (err) {
        console.log(err);
      }
    },
    async resetPassword() {
      try {
        this.loading = true;
        if (this.password == this.password2) {
          await this.$axios.$post("password/reset/confirm/", {
            password: this.password,
            token: this.token,
          });
          this.$router.push("/login");
        } else {
          this.$bvToast.toast("Password fields did not match", {
            title: "Error while reseting password",
            autoHideDelay: 5000,
            toaster: "b-toaster-top-center",
            variant: "danger",
          });
        }
      } catch (err) {
        console.log(err.response.status);
        this.$bvToast.toast(
          err.response.data.password || "An error occured",
          {
            title: "Error while reseting password",
            autoHideDelay: 5000,
            toaster: "b-toaster-top-center",
            variant: "danger",
          }
        );
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

Password reset is now setup. When use request a password, they will be sent a link by email that will let them reset their password!

## 7) Recipes pages

Let's create the logic to display, add, edit and delete the content of our app!

```bash
touch pwa/pages/recipes/index.vue
touch pwa/pages/recipes/add.vue
touch pwa/pages/recipes/_recipe/index.vue
touch pwa/pages/recipes/_recipe/edit.vue
touch pwa/pages/recipes/_recipe/delete.vue
```

First, we are going to implement the Vuez store logic by editing the `recipes.js` inside the store directory.

```javascript
### pwa/store/recipes.js
import Vue from "vue";

export const state = () => ({
  recipes: [],
});

export const mutations = {
  setRecipes(state, payload) {
    state.recipes = payload;
  },

  editRecipe(state, payload) {
    const index = state.recipes.findIndex((recipe) => recipe.id === payload.id);
    Vue.set(state.recipes, index, payload);
  },

  addRecipe(state, payload) {
    state.recipes.push(payload);
  },

  addRecipe(state, payload) {
    state.recipes.push(payload);
  },

  deleteRecipe(state, payload) {
    const index = state.recipes.findIndex((recipe) => recipe.id === payload);
    state.recipes.splice(index, 1);
  },
};

export const actions = {
  async getRecipes(context) {
    const response = await this.$axios.$get("recipes/");
    if (process.server) {
      response.forEach((recipe) => {
        recipe.image = recipe.image.replace(
          this.$config.axios.baseURL,
          this.$config.axios.browserBaseURL
        );
      });
    }
    context.commit("setRecipes", response);
  },

  async addRecipe(context, payload) {
    let formData = new FormData();
    formData.append("name", payload.name);
    if (!!payload.image) {
      formData.append("image", payload.image);
    }
    formData.append("instruction", payload.instruction);
    const response = await this.$axios.$post("recipes/", formData, {
      headers: {
        "Content-Type": "multipart/form-data",
      },
    });
    context.commit("addRecipe", response);
    return response;
  },

  async editRecipe(context, payload) {
    let formData = new FormData();
    formData.append("name", payload.name);
    if (!!payload.image) {
      formData.append("image", payload.image);
    }
    formData.append("instruction", payload.instruction);
    const response = await this.$axios.$patch(
      `recipes/${payload.id}/`,
      formData,
      {
        headers: {
          "Content-Type": "multipart/form-data",
        },
      }
    );
    context.commit("editRecipe", response);
    return response;
  },

  async deleteRecipe(context, payload) {
    const response = await this.$axios.$delete(`recipes/${payload}/`);
    context.commit("deleteRecipe", payload);
    return response;
  },
};

export const getters = {
  recipes(state) {
    return state.recipes;
  },
};
```

Note that we have to convert the image url sent on the server side to use the `browserBaseURL` instead of the `baseURL`. If we do not, our image are not going to be displayed properly.

Then, we can create our pages.

```vue
### pwa/pages/recipes/index.vue
<template>
  <div>
    <p>
      <big><strong>Browse awesome recipes!</strong></big>
    </p>
    <b-row>
      <b-col cols="12" sm="6" md="4" lg="3" xl="2"
        ><b-button to="/recipes/add" block class="mb-2">Add a new recipe</b-button>
      </b-col>
    </b-row>
    <b-row>
      <b-col
        v-for="recipe in recipes"
        :key="recipe.id"
        sm="6"
        lg="4"
        class="mb-4"
      >
        <NuxtLink :to="`/recipes/${recipe.id}`" class="text-dark">
          <b-card :to="`/${recipe.id}`" class="h-100">
            <b-img
              thumbnail
              rounded
              :src="recipe.image"
              :alt="`image-${recipe.name}`"
              style="width: 100%; height: 150px; object-fit: cover"
            />
            <h1>{{ recipe.name }}</h1>
          </b-card>
        </NuxtLink>
      </b-col>
    </b-row>
  </div>
</template>

<script>
export default {
  async fetch() {
    await this.fetchRecipes();
  },
  computed: {
    recipes() {
      return this.$store.getters["recipes/recipes"];
    },
  },
  methods: {
    async fetchRecipes() {
      await this.$store.dispatch("recipes/getRecipes");
    },
  },
};
</script>
```

```vue
### pwa/pages/recipes/add.vue
<template>
  <b-form @submit.prevent="addRecipe">
    <h1>Add a new recipe</h1>
    <b-form-group label="Name:">
      <b-form-input v-model="name" trim required></b-form-input>
    </b-form-group>
    <b-form-group label="Photo:">
      <b-form-file v-model="image" accept="image/*" required></b-form-file>
    </b-form-group>
    <b-form-group label="Instructions:">
      <b-form-textarea
        id="textarea"
        v-model="instruction"
        placeholder="Enter instructions..."
        rows="3"
        max-rows="20"
        trim
        required
      ></b-form-textarea>
    </b-form-group>
    <b-row class="justify-content-center">
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button to="/recipes" block>Cancel</b-button>
      </b-col>
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button type="submit" block variant="success">Add</b-button>
      </b-col>
    </b-row>
  </b-form>
</template>

<script>
export default {
  data() {
    return {
      name: null,
      image: null,
      instruction: null,
    };
  },
  methods: {
    async addRecipe() {
      try {
        const recipe = await this.$store.dispatch("recipes/addRecipe", {
          name: this.name,
          image: this.image,
          instruction: this.instruction,
        });
        this.$router.push(`/recipes/${recipe.id}`);
      } catch (err) {
        console.log(err);
        this.$bvToast.toast("An error occured while adding your recipe", {
          title: "Error",
          autoHideDelay: 5000,
          appendToast: false,
          toaster: "b-toaster-top-center",
          solid: true,
          variant: "danger",
        });
      }
    },
  },
};
</script>
```

```vue
### pwa/pages/recipes/_recipe/index.vue
<template>
  <div>
    <b-button to="/recipes" block class="d-sm-none mb-2"
      >view all recipes</b-button
    >
    <b-img
      thumbnail
      rounded
      :src="recipe.image"
      :alt="`image-${recipe.name}`"
      style="width: 100%; height: 250px; object-fit: cover"
    />
    <h1>{{ recipe.name }}</h1>
    <b-row v-if="recipe.user.id === $auth.user.id">
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button :to="`/recipes/${recipeId}/edit`" block variant="warning"
          >Edit</b-button
        >
      </b-col>
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button :to="`/recipes/${recipeId}/delete`" block variant="danger"
          >Delete</b-button
        >
      </b-col>
    </b-row>
    <h2>
      <small
        >Created
        {{
          new Date(recipe.created_at).toLocaleDateString("en", {
            year: "numeric",
            month: "long",
            day: "numeric",
          })
        }}</small
      >
    </h2>
    <h2>
      <small
        >Updated
        {{
          new Date(recipe.updated_at).toLocaleDateString("en", {
            year: "numeric",
            month: "long",
            day: "numeric",
          })
        }}</small
      >
    </h2>
    <p>
      <big
        ><strong>By {{ recipe.user.email }}</strong></big
      >
    </p>
    <p style="white-space: pre-line">
      {{ recipe.instruction }}
    </p>
  </div>
</template>

<script>
export default {
  async asyncData({ params, store, error }) {
    await store.dispatch("recipes/getRecipes");
    const recipe = store.getters["recipes/recipes"].find(
      (recipe) => recipe.id === parseInt(params.recipe)
    );
    if (!recipe) {
      error({ statusCode: 404, message: "Page not found" });
      return
    }
    return recipe;
  },
  computed: {
    recipeId() {
      return parseInt(
        this.$route.path.split("/")[this.$route.path.split("/").length - 1]
      );
    },
    recipe() {
      return this.$store.getters["recipes/recipes"].find(
        (recipe) => recipe.id === this.recipeId
      );
    },
  },
  methods: {
    async fetchRecipes() {
      await this.$store.dispatch("recipes/getRecipes");
    },
  },
};
</script>
```

```vue
### pwa/pages/recipes/_recipe/edit.vue
<template>
  <b-form @submit.prevent="editRecipe">
    <h1>Edit recipe</h1>
    <b-form-group label="Name:">
      <b-form-input
        v-model="name"
        placeholder="Enter a name ..."
        trim
        required
      ></b-form-input>
    </b-form-group>
    <b-form-group label="Photo:">
      <b-form-file v-model="image" accept="image/*"></b-form-file>
    </b-form-group>
    <b-form-group label="instruction:">
      <b-form-textarea
        id="textarea"
        v-model="instruction"
        placeholder="Enter instruction..."
        rows="3"
        max-rows="20"
        trim
        required
      ></b-form-textarea>
    </b-form-group>
    <b-row class="justify-content-center">
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button :to="`/recipes/${recipeId}`" block>Cancel</b-button>
      </b-col>
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button type="submit" block variant="warning">Edit</b-button>
      </b-col>
    </b-row>
  </b-form>
</template>

<script>
export default {
  async asyncData({ params, store, $auth, error }) {
    await store.dispatch("recipes/getRecipes");
    const recipe = store.getters["recipes/recipes"].find(
      (recipe) => recipe.id === parseInt(params.recipe)
    );
    if (!recipe) {
      error({ statusCode: 404, message: "Page not found" });
      return
    }
    if (recipe.user.id !== $auth.user.id ) {
      error({ statusCode: 403, message: "You do not have access to this resource" });
      return
    }
    const name = recipe.name;
    const instruction = recipe.instruction;
    return {name, instruction};
  },
  data() {
    return {
      name: null,
      image: null,
      instruction: null,
    };
  },
  computed: {
    recipeId() {
      return parseInt(
        this.$route.path.split("/")[this.$route.path.split("/").length - 2]
      );
    },
    recipe() {
      return this.$store.getters["recipes/recipes"].find(
        (recipe) => recipe.id === this.recipeId
      );
    },
  },
  methods: {
    async fetchRecipes() {
      await this.$store.dispatch("recipes/getRecipes");
    },
    async editRecipe() {
      try {
        await this.$store.dispatch("recipes/editRecipe", {
          id: this.recipeId,
          name: this.name,
          image: this.image,
          instruction: this.instruction,
        });
        this.$router.push(`/recipes/${this.recipeId}`);
      } catch (err) {
        console.log(err);
        this.$bvToast.toast("An error occured while editing your recipe", {
          title: "Error",
          autoHideDelay: 5000,
          appendToast: false,
          toaster: "b-toaster-top-center",
          solid: true,
          variant: "danger",
        });
      }
    },
  },
};
</script>
```

```vue
### pwa/pages/recipes/_recipe/delete.vue
<template>
  <b-form @submit.prevent="deleteRecipe">
    <p class="text-danger text-center">
      <big
        ><strong
          >You are about to delete this recipe, are you sure?</strong
        ></big
      >
    </p>
    <b-row class="justify-content-center">
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button :to="`/recipes/${recipeId}`" block>Cancel</b-button>
      </b-col>
      <b-col cols="6" sm="4" md="3" lg="2">
        <b-button type="submit" block variant="danger">Delete</b-button>
      </b-col>
    </b-row>
  </b-form>
</template>

<script>
export default {
  async fetch() {
    if (
      !this.$store.getters["recipes/recipes"].find(
        (recipe) => recipe.id === this.recipeId
      )
    ) {
      await this.fetchRecipes();
    }
  },
  computed: {
    recipeId() {
      return parseInt(
        this.$route.path.split("/")[this.$route.path.split("/").length - 2]
      );
    },
  },
  methods: {
    async fetchRecipes() {
      await this.$store.dispatch("recipes/getRecipes");
      if (!this.recipe) {
        this.$nuxt.error({
          statusCode: 404,
          message: `Recipe id ${this.recipeId} does not exists`,
        });
      }
    },
    async deleteRecipe() {
      try {
        await this.$store.dispatch("recipes/deleteRecipe", this.recipeId);
        this.$router.push("/recipes");
      } catch (err) {
        console.log(err);
        this.$bvToast.toast("An error occured while deleting your recipe", {
          title: "Error",
          autoHideDelay: 5000,
          appendToast: false,
          toaster: "b-toaster-top-center",
          solid: true,
          variant: "danger",
        });
      }
    },
  },
};
</script>
```

We are finaly done with the logic of our app

## 8) Configure PWA settings

In this section we are going to make some change to our `nuxt.config.js` to improve SEO and set up PWA.

```javascript
### pwa/nuxt.config.js
...
head: {
  title: "Awesome Recipes!",
  meta: [
    { charset: "utf-8" },
    { name: "viewport", content: "width=device-width, initial-scale=1" },
    {
      hid: "description",
      name: "description",
      content: "Browse the most delicious recipes",
    },
    {
      hid: "keyword",
      name: "keyword",
      content:
        "Recipes",
    },
    { name: "format-detection", content: "telephone=no" },
  ],
  link: [{ rel: "icon", type: "image/x-icon", href: "/favicon.ico" }],
},
...
pwa: {
    manifest: {
      lang: "en",
      name: "Awesome Recipes!",
      short_name: "Awesome Recipes!"
    },
  },
...
```

## Conclusion

Our Nuxt PWA is now setup!

In the [Part 3](https://blog.florianbgt.com/Universal_PWA_SSR_Nuxt_Django_docker_AWS_Part3_AWS_deployment) of this tutorial, we will see how to deploy our app to the cloud using AWS.

You can find the source code of this article on my github: https://github.com/florianbgt/Universal_PWA_SSR_Nuxt_Django_Docker_AWS

If you have any question or just want to chat, feel free to email me florian.bigot321@gmail.com