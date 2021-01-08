:::tip Это руководство было написано для Vue.js 3 и Vue Test Utils v2.
Версия для Vue.js 2 [здесь](/ru).
:::

## Тестирование Vuex в компонентах

Исходный код для теста на этой странице можно найти [здесь](https://github.com/lmiller1990/vue-testing-handbook/tree/master/demo-app-vue-3/tests/unit/ComponentWithVuex.spec.js).


## Использование `global.plugins` для тестирования `$store.state`

В обычном Vue приложении мы устанавливаем Vuex, используя `app.use(store)`, который делает хранилище доступным глобально. В модульном тесте мы делаем то же самое. Однако в отличие от обычного Vue приложения, мы не хотим использовать одно и то же хранилище во всех тестах – мы хотим каждый раз получать новую копию. Давайте посмотрим, как это сделать. Сначала сделаем простой компонент `<ComponentWithGetters>`, который отрисовывает имя пользователя, основываясь на состоянии хранилища.

```html
<template>
  <div>
    <div class="username">
      {{ username }}
    </div>
  </div>
</template>

<script>
export default {
  name: "ComponentWithVuex",

  data() {
    return {
      username: this.$store.state.username
    }
  }
}
</script>
```

Мы можем использовать `createStore` для создания Vuex хранилища. Затем мы передаём новый `store` в опцию монтирования компонента `global.plugins`. Полный тест выглядит так:

```js
import { createStore } from "vuex"
import { mount } from "@vue/test-utils"
import ComponentWithVuex from "@/components/ComponentWithVuex.vue"

const store = createStore({
  state() {
    return {
      username: "alice",
      firstName: "Алиса",
      lastName: "До"
    }
  },

  getters: {
    fullname: (state) => state.firstName + " " + state.lastName
  }
})

describe("ComponentWithVuex", () => {
  it("отрисовывает имя пользователя из настоящего Vuex хранилища", () => {
    const wrapper = mount(ComponentWithVuex, {
      global: {
        plugins: [store]
      }
    })

    expect(wrapper.find(".username").text()).toBe("Алиса")
  })
})
```

Тест проходит проверку. Создание нового Vuex хранилища вводит некоторый шаблонный код, из-за чего тест длится дольше. Если у вас много компонентов, который используют Vuex хранилище, то альтернативой будет использование опции монтирования `global.mocks` с подменой хранилища.

## Использование замоканного хранилища

Используя опцию монтирования `mocks`, вы можете замокать глобальный объект `$store`. Это значит, что вам не нужно создавать новое Vuex хранилище. Перепишем тест с использованием этой техники:

```js
it("отрисовывает имя пользователя, используя замоканное хранилище", () => {
  const wrapper = mount(ComponentWithVuex, {
    mocks: {
      $store: {
        state: { username: "Алиса" }
      }
    }
  })

  expect(wrapper.find(".username").text()).toBe("Алиса")
})
```

Я не рекомендую ни одно, ни другое. В первом тесте используется реальное хранилище Vuex, поэтому он больше отражает, как ваше приложение будет работать в реальной среде. Однако это порождает много шаблонного кода, и если у вас очень сложное хранилище Vuex, вы получите большие вспомогательные методы создания хранилища, которые затрудняют понимание тестов.

Второй подход использует замоканное хранилище. Хорошо здесь то, что все необходимые данные объявляются внутри теста, что делает его более понятным и компактным. Однако вы можете удалить всё хранилище Vuex, и этот тест все равно пройдёт проверку – не идеально.

Оба метода полезны, и ни один из них не лучше и не хуже другого.

## Тестирование `getters`

Используя вышеупомянутые техники, `getters` достаточно просто тестировать. Сначала сделаем компонент для теста:

```html
<template>
  <div class="fullname">
    {{ fullname }}
  </div>
</template>

<script>
export default {
  name: "ComponentWithGetters",

  computed: {
    fullname() {
      return this.$store.getters.fullname
    }
  }
}
</script>
```

Мы хотим проверить, что компонент правильно отрисовывает `fullname` пользователя. Для этого теста нам не важно откуда приходит `fullname`: важно только то – правильно ли оно отрисуется.

Сначала используем настоящее Vuex хранилище. Тест будет выглядеть так:

```js
const store = new createStore({
  state: {
    firstName: "Алиса",
    lastName: "Доу"
  },

  getters: {
    fullname: (state) => state.firstName + " " + state.lastName
  }
})

it("отрисовывает имя пользователя, используя настоящий геттер Vuex", () => {
  const wrapper = mount(ComponentWithGetters, {
    global: {
      plugins: [store]
    }
  })

  expect(wrapper.find(".fullname").text()).toBe("Алиса Доу")
})
```

Тест очень компактный – всего две строчки кода. Тем не менее, здесь много настроек – мы перестраиваем хранилище Vuex. Обратите внимание, что мы *не* используем хранилище Vuex, которое будет использовать наше приложение, мы создали минимальное хранилище с базовыми данными, необходимыми для геттера `fullname`, которого ожидал компонент.

Как альтернативу, можно импортировать настоящее хранилище Vuex с реальным геттером. Это вводит в тест ещё одну зависимость, и при разработке большой системы возможно так, что хранилище Vuex может быть разработано другим программистом и ещё не реализовано.

Давайте посмотрим, как мы можем написать тест, используя опцию монтирования `global.mocks`.

```js
it("отрисовываем имя пользователя, используя вычисленные опции монтирования", () => {
  const wrapper = mount(ComponentWithGetters, {
    global: {
      mocks: {
        $store: {
          getters: {
            fullname: "Алиса До"
          }
        }
      }
    }
  })

  expect(wrapper.find(".fullname").text()).toBe("Алиса До")
})
```

Теперь вся необходимая информация содержится в тесте. Отлично! Наш тест полноценный - в нём есть всё для понимания работы тестируемого компонента.

## Помощники `mapState` и `mapGetters` 

Техники, описанные выше, также работают вместе с помощниками `mapState` и `mapGetters`. Мы можем обновить  `ComponentWithGetters` вот так:

```js
import { mapGetters } from "vuex"

export default {
  name: "ComponentWithGetters",

  computed: {
    ...mapGetters([
      'fullname'
    ])
  }
}
```

Тест все ещё проходит проверку.

## Заключение

В этом руководстве обсудили:

- как использовать `createStore` для создания настоящие Vuex хранилища и устанавливать его с помощью `global.plugins`
- как тестировать `$store.state` и `getters`
- как использовать `global.mocks` в опциях монтирования, чтобы устанавливать желаемое значение для `$store.state` или `getters`

Техники для тестирования реализации Vuex геттеров в изоляции можно найти в [этом руководстве](https://lmiller1990.github.io/vue-testing-handbook/v3/ru/vuex-getters.html).

Исходный код для теста на этой странице можно найти [здесь](https://github.com/lmiller1990/vue-testing-handbook/tree/master/demo-app-vue-3/tests/unit/ComponentWithVuex.spec.js).
