# Vue custom input

Most of us have faced it: build a custom input component. There are multiple reasons behind it but in general, it has custom styles and we should be able to reuse it.

Although it might sound simple it has some gotchas and from time to time we end up going through the documentation to check some details. It gets a bit more complicated if you're not that familiar with few Vue concepts.

Last month, February 2021, it happened again. When possible I try to help people in a Vue Slack group and this question popped up once again. Not exactly this question but the user had issues building a custom input component. The problem was related to some concepts.

To consolidate this knowledge for myself, and use it as some sort of documentation for others, I decided to wrap up the process of writing a custom input.

## v-model and `<input>`

Once we start building forms with Vue we learn the directive `v-model`. It does a lot of the hard work for us: it binds a value to an input. It means that whenever we change the input's value the variable will also be updated.

The official docs do a great job explaining how it works: https://vuejs.org/v2/guide/forms.html

In short we can have the following template and we're fine:

```html
<!-- UsernameInput.vue -->
<template>
  <label>
    Username
    <input type="text" name="username" v-model="username">
  </label>
</template>

<script>
export default {
  name: 'UsernameInput',
  data() {
    return {
      username: 'Initial value',
    };
  },
}
</script>
```

We'll have an input that has `Initial value` as the initial value and the username data will be automatically updated once we change the input's value.

The problem with the above component is that we can't reuse it. Imagine we've a page where we need the username and the e-mail, the above component won't handle the e-mail case as the data is inside the component itself, not somewhere else (like the parent component, for example). That is where custom input components shine and also one of its challenges: keep the `v-model` behavior consistent.

## The wrong custom input component

Well, why am I showing this example? The answer is: this is the first approach most of us will try.

Let's see how we are going to use our custom input component:

```html
<!-- App.vue -->
<template>
  <custom-input :label="label" v-model="model" />
</template>

<script>
import CustomInput from './components/CustomInput.ue';

export default {
  name: 'App',
  components: { CustomInput },
  data() {
    return {
      label: 'Username',
      model: '',
    };
  },
}
</script>
```

The custom input expects a `label` and a `v-model` in this case and will look like the component below:

```html
<!-- CustomInput.vue -->
<template>
  <label>
    {{ label }}
    <input type="text" :name="name" v-model="value" />
  </label>
</template>

<script>
export default {
  name: 'CustomInput',
  props: {
    label: {
      type: String,
      required: true,
    },
    value: {
      type: String,
      required: true,
    },
  },
  computed: {
    name() {
      return this.label.toLowerCase();
    },
  },
}
</script>
```

First, it expects the `label` as property and computes the `name` on top of that (it could also be a property). Second, it expects a `value` property and binds it to the `<input>` through `v-model`. The reason behind that can be found [in the docs](https://vuejs.org/v2/guide/components.html#Using-v-model-on-Components) but in short, when we use `v-model` in a custom component it will get `value` as a property which is the value from the `v-model` variable used. In our example, it'll be the value from `model` defined in `App.vue`.

If we try the code above, it'll work as expected, but why is it wrong? If we open the console we will see something like this:

```
[Vue warn]: Avoid mutating a prop directly since the value will be overwritten whenever the parent component re-renders. Instead, use a data or computed property based on the prop's value. Prop being mutated: "value"
```

It complains that we're mutating a property. The way Vue works is: the child component has props that come from the parent component and the child component emits changes to the parent component. Using `v-model` with the `value` prop that we got from the parent component violates it.

Another way to see this issue is rewriting the `App.vue` like this:

```html
<!-- App.vue -->
<template>
  <custom-input :label="label" :value="model" />
</template>

...
```

The main difference is using `:value` instead of `v-model`. In this case, we're just passing `model` to the `value` property. The example still works and we get the same message in the console.

The next step is to rework the example above and make sure it works as expected.

## The happy custom input component

The happy custom input component doesn't mutate its prop but emits the changes to the parent component.

[The docs](https://vuejs.org/v2/guide/components.html#Using-v-model-on-Components) have this exact example but we'll go a bit further here. If we follow the docs, our `CustomInput` should look like the one below:

```html
<!-- CustomInput.vue -->
<template>
  <label>
    {{ label }}
    <input type="text" :name="name" :value="value" @input="$emit('input', $event.target.value)" />
  </label>
</template>

<script>
export default {
  name: 'CustomInput',
  props: {
    label: {
      type: String,
      required: true,
    },
    value: {
      type: String,
      required: true,
    },
  },
  computed: {
    name() {
      return this.label.toLowerCase();
    },
  },
}
</script>
```

This is enough to make it work. We can even test it against both `App.vue`, the one using `v-model`, where everything works as expected, and the one using `:value` only, where it doesn't work anymore as we stopped mutating the property.

### Adding validation (or operation on data change)

In case we need to do something when the data changes, for example checking if it is empty and show an error message, we need to extract the emit. We'll have the following changes to our component:

```html
<!-- CustomInput.vue -->
<template>
...
    <input type="text" :name="name" :value="value" @input="onInput" />
...
</template>

<script>
...
  methods: {
    onInput(event) {
      this.$emit('input', event.target.value);
    }
  }
...
</script>
```

Now we add the empty check:

```html
<!-- CustomInput.vue -->
<template>
...
    <p v-if="error">{{ error }}</p>
...
</template>

<script>
...
  data() {
    return {
      error: '',
    };
  },
...
    onInput(event) {
      const value = event.target.value;

      if (!value) {
        this.error = 'Value should not be empty';
      }

      this.$emit('input', event.target.value)
    }
...
</script>
```

It kind of works, first it doesn't show any errors and if we type then delete it'll show the error message. The problem is that the error message never disappears. To fix that we need to add a watcher to the value property and clean the error message whenever it is updated.

```html
<!-- CustomInput.vue -->
...
<script>
...
  watch: {
    value: {
      handler(value) {
        if (value) {
          this.error = '';
        }
      },
    },
  },
...
</script>
```

We could achieve a similar result adding an `else` inside `onInput`. Using the watcher enables us to validate before the user updates the input value, if desirable.

If we add more things we most probably will be expanding this component even more and things are spread all over the `<script>` block. To group things a bit we can try a different approach: use computed together with `v-model`.

### Combining computed and `v-model`

Instead of listening to the `input` event and then emitting it again, we can leverage the power of `v-model` and `computed`. It is the closest we can get to the wrong approach but still make it right 😅
Let's rewrite our component like that:

```html
<!-- CustomInput.vue -->
<template>
...
    <input type="text" :name="name" v-model="model" />
...
</template>

<script>
...
  computed: {
    ...
    model: {
      get() {
        return this.value;
      },
      set(value) {
        this.$emit('input', value);
      },
    },
  },
...
</script>
```

We can get rid of the `onInput` method and also from the watcher as we can handle everything within `get/set` functions from the computed property.

One cool thing we can achieve with that is the usage of modifiers, like `.trim/number` that would need to be manually written before.

This is a good approach for simple input components. Things can get a bit more complex and this approach doesn't fulfill all the use cases, if that is the case we need to go for binding value and listening to events. A good example is if you want to support the `.lazy` modifier in the parent component, you will need to manually listen to `input` and `change`.

## Extra: the `model` property

The [`model` property](https://vuejs.org/v2/api/#model) allows you to customize a bit the `v-model` behavior. You can specify which property will be mapped, the default is `value`, and which event will be emitted, the default is `input` or `change` when `.lazy` is used.

This is especially useful if you want to use the `value` prop for something else, as it might make more sense for a specific context, or just want to make things more explicit and rename `value` to `model`, for example. In most cases, we might use it to customize checkboxes/radios when getting objects as input.

## So what?

My take comes from how complex your custom input needs to be:

- It was created to centralized the styles in one component and its API is pretty much on top of Vue's API: `computed` + `v-model`. It falls pretty much on our example, it has simple props and no complex validation.

```html
<!-- CustomInput.vue -->
<template>
  <label>
    {{ label }}
    <input type="text" :name="name" v-model="model" />
  </label>
</template>

<script>
export default {
  name: 'CustomInput',
  props: {
    label: {
      type: String,
      required: true,
    },
    value: {
      type: String,
      required: true,
    },
  },
  computed: {
    name() {
      return this.label.toLowerCase();
    },
    model: {
      get() {
        return this.value;
      },
      set(value) {
        this.$emit('input', value);
      },
    },
  },
}
</script>
```

- Everything else (which means that you need to tweak a lot the previous setup to support what you need): listeners, watchers and whatelse you might need. It might have multiple states (think of async validation where a loading state might be useful) or you want to support `.lazy` modifier from the parent component, are good examples to avoid the first approach.

```html
<!-- CustomInput.vue -->
<template>
  <label>
    {{ label }}
    <input type="text" :name="name" :value="value" @input="onInput" @change="onChange" />
  </label>
</template>

<script>
export default {
  name: 'CustomInput',
  props: {
    label: {
      type: String,
      required: true,
    },
    value: {
      type: String,
      required: true,
    },
  },
  /* Can add validation here
  watch: {
    value: {
      handler(newValue, oldValue) {

      },
    },
  }, */
  computed: {
    name() {
      return this.label.toLowerCase();
    },
  },
  methods: {
    onInput(event) {
      // Can add validation here
      this.$emit('input', event.target.value);
    },
    onChange(event) { // Supports .lazy
      // Can add validation here
      this.$emit('change', event.target.value);
    },
  },
}
</script>
```