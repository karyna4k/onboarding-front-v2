---
language: ru
title: Введение
---
# Начнём!

Мы пишем фронтенд, используя такой стек:
- Vue 2
- Nuxt 2
- Typescript
- tailwindcss
- scss

В скором будущем перейдём на Vue 3 и Nuxt 3, для этого появится отдельное введение вроде этого.

## Архитектура

В построении приложений придерживается принципов Clean Architecture. Насколько понимаю, всё пошло [примерно отсюда](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html).
Что же касается такой архитектуры именно в контексте Vue, хорошая серия статей есть на Medium:

- [Введение](https://javascript.plainenglish.io/building-vue-enterprise-application-part-0-overture-6d41bea14236)
- [Сущности-модели](https://levelup.gitconnected.com/building-vue-enterprise-application-part-1-entities-808077f3d2e7)
- [Сервисы](https://javascript.plainenglish.io/building-vue-enterprise-application-part-2-services-f7ec400190e7)
- [Стор](https://itnext.io/building-vue-enterprise-application-part-3-the-store-dbda0e4bb117)
- [UI](https://itnext.io/building-vue-enterprise-application-part-4-ui-components-21a45b3067a4)

Читать всё не обязательно, но пролистать точно будет полезно. Если в двух словах - всё о разделении кода UI компонентов, логики и взаимодействия с АПИ.

<img src="/img/clean-architecture.png" alt="Clean Architecture" style="width:400px;margin:auto;"/>

У нас типичный сценарий использования этого великолепия такой. Скажем, в компоненте нужно отобразить лист каких-то сущностей. Для сущности в `~/models` должны быть определены:
1. **тип** данных, приходящих от сервера;
1. **интерфейс**, который мы будем использовать непосредственно в аппе;
1. **класс**, реализующий этот интерфейс, принимающий в конструкторе АПИ-дату и формирующий из неё необходимые поля. Помимо полей, он может включать геттеры и методы.

Чтобы получить данные, компонент вызывает экшн стора. Тот, в свою очередь, вызывает метод сервиса, обслуживающий нужный метод/url АПИ. Из сервиса в стор возвращается дата от сервера.

Стор маппит полученный результат в упомянутые выше классы и пробрасывает в компонент готовые экземпляры. При необходимости, стор сохраняет результат в свой стейт (например, при аутентификации или получении любых других данных, которые используются в разных не сильно связанных частях аппа).

Чуть подробнее о каждом из упомянутых здесь модулей - ниже.

### ~/models

За каждую сущность отвечает отдельная директория. Если сущность относится к какой-то конкретной "родительской" сущности, лучше создать и вложенную директорию.

###### \<entity\>.types.ts

Здесь определены тип данных, получаемых по АПИ, и интерфейс класса для аппа. Например, для сущности Работы:

```ts
export type ItemData = {
  id: ItemKey
  title: string
  description?: string
  collections?: CollectionData[]
  file?: MediaData
}
```

Интерфейс класса, который используем в аппе, расширяет и переопределяет этот тип. Важен и "апгрейд" типа вложенных сущностей с _даты-с-сервера_ до _интерфейса-созданного-класса_.

```ts
export interface IItemModel extends ItemData {
  collections: ICollectionModel[]
  file: IMediaModel
}
```

Также здесь были бы всевозможные `enum` и другие типы для отдельных полей.

###### \<entity\>.ts

В соседнем файле - класс, который будет использоваться дальше в компоненте. Основное его назначение - взять из ответа только те данные, которые нужны, и привести их к подходящему виду.

```ts
export class ItemModel implements IItemModel {
  id: ItemKey
  title: string
  description?: string
  collections: ICollectionModel[]
  file: IMediaModel

  constructor(data: ItemData) {
    this.id = data.id
    this.title = data.title
    this.description = data.description
    this.collections = data.collections
      ?.map(collection => new CollectionModel(collection)) || []
    this.file = new MediaModel(data.file)
  }
}
```

> Не стоит слишком увлекаться вынесением геттеров в класс: это чистый typescript вне vue, а значит, не реактивный. Лучше создавать их непосредственно в компонентах.

Важно не забыть маппить вложенные модели (`this.collections` выше), чтобы обработать и их, а не сохранить просто сырой ответ бэка.

Также здесь - какие-то вспомогательные `const` вроде валидационных ограничений, связанных с сущностью.

--- 

При необходимости в директорию добавится и файл тестов `<entity>.spec.ts`.

Не забудь экспортировать модели через `index.ts`, чтобы чтобы можно было импортировать модели из рутовой папки:
```ts
import { MyData, IMyModel, MyModel } from '~/models'
```

### ~/services

Сервисы можно разделить на два типа:
- **CRUD** (Create Read Update Delete);
- **кастомные**.

Вторые - любые изощрённые операции вроде работы с буфером, сокетами etc. Для них пишем просто классы с нуля, в зависимости от нужд.

Первые же - для дёргания АПИ, а значит это что-то более систематическое. Методы, например, создания, чтения и апдейта двух разных сущностей работают очень схоже. Поэтому для CRUD сервисов мы создали некоторые абстрактные классы. Их можно пощупать в `~/services/core/crud/crud.ts`.

Благодаря ним можно очень быстро создать АПИ-сервис, просто правильно параметризовав классы.

```ts
export class CollectionsService extends ApiReadService<
CollectionData,
CollectionKey
> {}
```

Если добавляется какой-то менее дженериковый функционал, можно просто расширить эти классы.

Провайдер в директории сервисов - функция, которая сделает сервисы доступными из `this` vue-компонентов и nuxt-контекста (о них - позже). В провайдере и создаются сервисы, получая на входе апп - часть урла АПИ, которую обслуживает сервис, - и nuxt-контекст:

```ts
$services = {
  items: new ItemsService('artwork', ctx),
}
```

Подобным способом создаём и вложенные сервисы, передавая в конструктор также и base урл родительского сервиса, чтобы "стакать" обслуживаемый урл.

```ts
constructor(protected app: string, ctx: Context, root?: string) {
  super(app, ctx, root)
  this.collections = new CollectionsService('collection', ctx, this.baseURL)
}
```

Такой манёвр, например, создаст сервис, работающий с урлом `<ctx.$axios.defaults.baseURL>/artwork/collection/`

В качестве http-клиента используем [axios](https://axios.nuxtjs.org/options).

### ~/store

В качестве стора используем [Vuex 3](https://vuex.vuejs.org). Он реализует flux-подобную архитектуру. В двух словах, стор имеет общее для всего аппа состояние (state), публичные экшны, доступные извне, и мутации, меняющие состояние, а также вложенные модули - по сути, такие же сторы.

> В уже написанных проектах вместо `~/store` используется `~/store-modules`. Это связано с тем, что `store` - особенная директория для Nuxt, и он создавал стор-модуль из каждого файла внутри неё, что не позволяло безнаказанно поместить туда структуру с core, моделями etc. 
>
> Такую обработку директории store можно отключить в конфиге накста, что и стали делать в новых проектах. Когда дойдут руки до рефакторинга - везде будем использовать именно store.

По аналогии с сервисами, сторы могут быть **CRUD** и **кастомные**.

Вторые существуют для обслуживания кастомных сервисов. Тот же стор аутентификации обычно включает ряд экшнов/мутаций для сохранения и апдейта данных профиля, ролей, JWT-токена и всего-всего.

> При написании кастомных экшнов нужно помнить, что Vuex экшн ожидает на входе только один аргумент - payload. Если ты определишь под `@Action` декоратором функцию, принимающую 2+ аргументов, все, кроме первого, останутся на морозе и будут равны `undefined` внутри экшна.

Для CRUD сторов также созданы абстрактные классы-заготовки (см. `~/store/core/crud/crud.ts`). Незамысловатые сторы можно быстро создать, параметризовав класс.

Наиболее полный класс (`ApiFullModule`) предоставляет `read`, `get`, `create`, `update` (patch под капотом), `replace` (put под капотом) и `delete` методы.

```ts
@Module({
  name: 'items',
  stateFactory: true,
  namespaced: true,
})
export class ItemsModule extends ApiReadWriteModule<ItemData, IItemModel, ItemKey> {
  get service() {
    return $services.items
  }

  get GetModel() {
    return ItemModel
  }

  get ReadModel() {
    return ItemModel
  }
}
```

Для параметризации здесь передаются тип даты от АПИ, интерфейс модели и ключ. Но вообще абстрактные классы позволяют использовать разные модели для Read и Get (по id) методов, а также параметризовать входные данные для Create и Update.

Сервис здесь - геттером, а не обычным полем, из-за особенностей видимости разных частей стора внутри экшнов. Если бы он был определён обычным полем, декоратор бы обработал его как часть стейта, и он был бы недоступен из экшна, так как менять стейт концептуально "уполномочены" только мутации.

> При переходе на vue3/nuxt3 в качестве стора, скорее всего, будет использоваться не [Vuex 4](https://next.vuex.vuejs.org), а [pinia](https://pinia.esm.dev/core-concepts/#using-the-store) - альтернативный стор от участника core-команды vue и vuex. Pinia легче, чаще и смелее обновляется, а также избавлен от мутаций, которые очень часто выглядели каким-то рудиментом и имели уж очень узкое и спорное предназначение.

## Vue + Typescript

Используем [Vue 2](https://vuejs.org/v2/api/) в связке с Typescript, поэтому на вид, можно сказать, ближе к 3-й версии. Нет Options API - есть класс, наследующийся от Vue.

```vue
<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator'

@Component
export default class MyComponent extends Vue {
  // ...
}
</script>
```

- Геттеры здесь = computed properties из vue.js. Можно использовать также get + set.
- Обычные переменные внутри класса = data.
- Методы класса = методы vue.js, а также хуки жизненного цикла вроде `mounted` или `created`.

Для более же vue-специфических вещей вроде props, watch, моделей и т.п. используются [декораторы](https://github.com/nuxt-community/nuxt-property-decorator#decorators-and-helpers).

Миксины можно применять, наследуя от них компоненты. По сути, они представляют собой только `script` часть обычного vue компонента.

```ts
@Component
export class MyMixin extends Vue {
  // ...
}
```

```ts
@Component
export class MyComponent extends MyMixin {
  // ...
}
```

Что же касается темплейтов, что теги компонентов используем в `kebab-case` (а не `CamelCase`, хотя так тоже работает). Это ближе к обычному html и смотрится органичнее. 🙂

---

Также замечу, что разбиение на максимально атомарные компоненты - это хорошо. Это уменьшает размер бандлов билда и повышает качество кода.

Что-то можно вынести отдельно, чтобы избежать дублирования? Что-то можно написать более универсально, чтобы не писать похожую логику? Сделаем же это!

`~/components/LazyList.vue` - как пример универсального компонента.


## Nuxt

Nuxt 2 - фреймворк над vue, делающий разработку проще. Обязательно чекни [документацию](https://nuxtjs.org/docs/get-started/directory-structure), там много полезных фич.

Имена компонентов в Nuxt определяются по их пути/имени файла внутри `~/components`. Это позволяет избегать *VeryLongComponentNames.vue* и добавляет порядка компонентам. Скажем, чтобы получить компоненты `item-card` и `item-card-skeleton`, можно создать такую структуру:

```
components/
└─ item/
    └─ card/
      └─ _.vue
      └─ Skeleton.vue
```

Компоненты будут доступны внутри других компонентов без необходимости их явно регистрировать.

Другие наиболее дерзкие фичи:
- Автогенерируемый роутер на основе структуры `~/pages` директории.
- Удобный инжект глобальных свойств через `~/plugins` (те же стор и сервисы внедряем как раз через плагины).
- Метод `asyncData`, позволяющий сделать запрос данных ещё до монтирования страницы, и в зависимости от результата уже разруливать дальнейшее поведение.
- ...и много чего ещё.

Для Nuxt также есть достаточно много полезных модулей вроде [интернационализации](https://i18n.nuxtjs.org), [обработки маркдаунов](https://go.nuxtjs.dev/content) или [работы с темами](https://color-mode.nuxtjs.org).

Это примеры модулей, которые используются в этом intro-проекте. Посмотреть, что вообще существует для Nuxt, можно [здесь](https://modules.nuxtjs.org).

### i18n

Так как мы делаем продукты для международной аудитории, важным моментом является интернационализация.

Месседжи для переводов записываются в json-подобных js файлах в директории `~/lang`.

Модуль поддерживает [плурализацию](https://kazupon.github.io/vue-i18n/guide/pluralization.html#accessing-the-number-via-the-pre-defined-argument):
```js
export default {
  items: {
    counter: 'Нет работ | 1 работа | {count} работы | {count} работ',
  },
}
```
```ts
this.$tc('items.counter', 1) // "1 работа"
this.$tc('items.counter', 18) // "18 работ"
this.$tc('items.counter', 42) // "18 работы"
```

Также можно передавать аргументы и ссылаться на другие месседжи внутри файла:

```js
export default {
  hello: 'привет',
  greater: '@.capitalize:hello, {name}.',
}
```
```ts
$t('greater', { name: 'username' }) // "Привет, username."
```

Модуль [i18n](https://i18n.nuxtjs.org) интегрирован в Nuxt, поэтому хорошо вклинивается в автогенерируемые роуты, внедряя префикс с кодом локали (например, `/ru/docs`).

Переходить на страницы с учётом локали можно с помощью специальных методов:

```vue
<nuxt-link :to="localePath({ name: 'docs' })">README</nuxt-link>
```

## UI kits

Методом проб и ошибок, остановились на [Buefy](https://buefy.org/documentation). Качественная библиотека, почти без неожиданностей. Под неё есть и [модуль](https://github.com/buefy/nuxt-buefy) для Nuxt.

Если интересно, мой личный топ UI библиотек для vue можно посмотреть [здесь]().

> Для vue 3 создатели Buefy выпустили отдельную библиотеку - [Oruga](https://oruga.io/documentation/#setup). Для vue 3 существует и другая очень многообещающая либа - [Naive UI](https://www.naiveui.com/en-US/dark/components/button), причём хорошая и по визуалу. Нужно будет попробовать.

Берём готовые стили либы, только если это внутренний проект или прототип.

Для публичных же проектов стили пишем исключительно свои. В конфиге того же `Buefy` на этот случай есть опция `css: false`.

Для валидации форм используем [VeeValidate](https://vee-validate.logaretm.com/v3).

## Стили

### scss

Стили пишем, используя [Sass](https://sass-lang.com), а точнее, его `scss` синтаксис.

Стили стараемся писать в соответствующем компоненте, но если что-то глобальное - идём в `~/assets/scss` и подключаем в лэйауте.

> Почему прямо в лэйауте, а не внутри `css` секции в `nuxt.config.js`? Чтобы при изменении стилей изменения подтягивались через Hot Reload без пресобирания всего билда с "новым" конфигом.
>
> `css` секция же хороша для подключения стилей зависимостей, например, пака иконок или UI библиотеки.

В `~/assets/scss/utils/_mixins.scss` доступны некоторые утилиты-миксины, делающие написание стилей чуть проще. Они доступны внутри vue компонентов без необходимости импорта за счёт подключения через [style-resources](https://github.com/nuxt-community/style-resources-module).

Вероятно, в скором времени они станут ненужными, учитывая, что мы расширяем использование `tailwindcss`, куда можно "переместить" все стилевые утилиты. О нём - дальше.

### tailwindcss

[Tailwind CSS](https://tailwindcss.com/docs/configuration) - utility-first фреймворк, содержащий множество классов на все случаи жизни (например, _rotate-45_, _my-12_, _font-black_ и т.п.).

Сам подход писать стили классами в html - очень сомнительный, как по мне (проблемы с дублированием и читаемостью темплейта). Но! Tailwind позволяет с помощью директивы `@apply` формировать из них новые стили.

```scss
.my-pretty-element {
  @apply tw-pt-6 tw-pb-12 tw-rounded;
}
```

Такая фича делает написание стилей, напротив, компактным и удобным. 

Сохраняется преимущества НЕ-utility подхода:
- **семантика** названий классов;
- **компактность** темплейта;
- **переиспользование** и поддержка,

...Но и добавляются фичи utility подхода:
- быстрое **прототипирование**;
- готовые **утилиты** (давай признаемся, мы и так сами писали классы-для-одного-свойства, ну или брали готовые из UI либ).

Хотя главные плюс, наверное, в крутом конфиге.

#### Кастомизация

Конфиг находится в `~/tailwind.config.js`.

Как можно было понять из примера выше, утилиты tailwind используем с префиксом `tw-` - чтобы стили не спорили со стилями UI либы, если те придётся подключить.

Дальше - [тема](https://tailwindcss.com/docs/theme). Если оставить пустой, подключится дефолтная тема tailwind (речь, кстати, про паддинги, шрифты и что угодно - не только про палитру). Чтобы переопределить доступные значения для утилит - добавить поле с названием утилиты; чтобы расширить - добавить поле внутри `extend` опции.

```js
module.exports = {
  theme: {
    opacity: { // override tw-opacity-* tw-bg-opacity-* tw-text-opacity-* etc
      0: 0,
    },
    extend: {
      screens: { // extend breakpoints
        'xs': '400px',
      },
    },
  },
}
```

Таким образом можно подстроить весь tailwind под свой дизайн. Это касается и цветовых тем.

Сами переменные для разных тем мы храним в `~/assets/scss/tailwind.scss`, они заданы через `:root` переменные.

```scss
:root {
  // Общая палитра
  --c-black: 0, 0, 0;
  --c-red: 255, 84, 94;

  // Светлая тема
  --c-color-base: var(--c-black);
  --c-bg-base: 255, 255, 255;
}
.dark-mode {
  // Тёмная тема
  --c-color-base: 240, 240, 250;
  --c-bg-base: 25, 25, 30;
}
```

> Лучше не использовать значения прямо из общей палитры для какого-то определённого компонента.
>
> Скажем, если текст ссылки в светлой и тёмной темах - c-red, это ещё не значит, что не появится третья тема cyberpunk, где ссылка будет другого цвета. Тогда лучше сразу создать переменную c-link, определить как `--c-link: var(--c-red);` для светлой и тёмной темы и ссылаться уже на неё.
>
> Вывод - переменные должны быть более утилитарные.

Эти переменные уже можно использовать в стилях вроде `rgb(var(--c-color-base))`, но лучше добавить их в `~/tailwind.config.js`, чтобы раскрыть все преимущества фреймворка.

Можно добавить

```js
module.exports = {
  theme: {
    textColor: {
      base: co('--c-color-base'),
    },
  },
}
```

> `co` здесь отдаёт функцию, возвращающую значение цвета с учётом входной переменной прозрачности. Как это понимать?
>
> В tailwind есть утилиты прозрачности текста, бэкграунда, бордера. В css же таких атрибутов нет, они - часть самого цвета. Эта прозрачность, идущая от tw-bg-opacity-50, и примешивается как 4-й параметр rgba(), первые же три идут от нашего цвета из темы. Именно поэтому цвета в теме - три канала (r, g и b), а не "готовые" hex и rgb.
>
> Прозрачность цветов по умолчанию - 1.

Существенный апгрейд по сравнению с тем, как мы делали раньше. Раньше мы использовали самописный scss миксин, который хитро "оборачивал" стили, подменяя глобальные переменные переменными из темы, а потом возвращая всё назад.
```scss
.my-fancy-block {
  @include themed() { color: theme('c-color-base'); }
}
```

Сейчас "темизация" сократилось, стала доступной для автодополнения и, что более важно, доступным для прототипирования прямо в темплейте.
```scss
.my-fancy-block {
  @apply tw-text-base;
}
```

Также в экосистеме tailwind доступны полезные плагины - например, `aspect-ratio` и `line-clamp`. Оба используются в этом проекте: первый обеспечивает квадратные пропорции превью работы; второй - ограничивает (через truncate) число строк названия и описания.

## Линтеры

Для линта кода используем два инструмента:
- [eslint](https://eslint.org/docs/user-guide/configuring/rules#configuring-rules) - для `vue`,`ts`,`js` файлов;
- [stylelint](https://stylelint.io/developer-guide/rules) - для всего остального.

Оба линта запускаются для staged файлов при попытке коммита. Обрабатывает это более легковесная альтернатива `husky` - [simple-git-hooks](https://github.com/toplenboren/simple-git-hooks).

Активируй их, запустив следующие команды:
```bash
# [Опционально] Если ты использовал(а) раньше husky
git config core.hooksPath .git/hooks/
rm -rf .git/hooks

# Обновить ./git/hooks
npx simple-git-hooks
```

## Пакетный менеджер

Наконец, в качестве пакетного менеджера используем [yarn](https://yarnpkg.com/getting-started/usage).

До недавнего времени использовали `npm`, но в как-то раз во вложенной зависимости `coa` [появилась уязвимость](https://www.bleepingcomputer.com/news/security/popular-coa-npm-library-hijacked-to-steal-user-passwords). Пришлось ограничить её версию, что было удобно сделать средствами yarn. Так и повелось.

А ещё говорят, он быстрее. Но этого я пока не заметил.

В скриптах внутри `~/package.json` используется `nr` (r - от "run"; для "install" будет `ni`, например). Это команда из [@antfu/ni](https://github.com/antfu/ni#nr---run) - небольшого пакета, который детектит `.lock` файл и вызывает нужный пакетный менеджер, пробрасывая туда входные параметры.

---

**Вау.**

Ты что, правда дочитал(а)?