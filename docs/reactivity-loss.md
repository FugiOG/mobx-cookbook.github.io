# Все сценарии потери реактивности

Компонент не обновляется после изменения стейта? Эта глава призвана разобрать все возможные сценарии потери
реактивности, поэтому тут вы точно должны найти ваш случай. Часть проблем больше связана с особенностями JS, нежели с
Mobx.

### Компонент не observer

Для работы реактивности компоненты должны подписываться на изменения и отписываться от них. Mobx не
требует [ручных подписок](https://rxjs.dev/guide/subscription#subscription) как RxJS, не требует использования хуков
вроде [useSelector](https://react-redux.js.org/api/hooks#useselector-examples) для извлечения отдельных частей стейта. С
Mobx достаточно оборачивать компоненты в observer. Взамен вам не нужно думать про вложенность, мемоизацию, селекторы и
подписки. Ещё использование observer не более затратно чем использование `React.memo`, который вы будете вынуждены
использовать для достижения такого же уровня производительности. Разберём примеры:

❌ Не будет работать, отсутствует подписка:

```typescript jsx
class Store {
  count = 0

  constructor() {
    makeAutoObservable(this)
  }

  increment() {
    this.count++
  }
}

const store = new Store()

const Counter = () => {
  return <button onClick={() => store.increment()}>Clicked times: {store.count}</button>
}
```

✅ Работает, так как есть observer:

```typescript jsx
class Store {
  count = 0

  constructor() {
    makeAutoObservable(this)
  }

  increment() {
    this.count++
  }
}

const store = new Store()

// Добавлен observer 👇
const Counter = observer(() => {
  return <button onClick={() => store.increment()}>Clicked times: {store.count}</button>
})
```

Если вам интересно как внутри устроен механизм подписок Mobx, то это в упрощённом виде описано в
главе [Mobx в 50 строчек кода](mobx-inside). У Mobx
есть [ESLint-правило](https://github.com/mobxjs/mobx/tree/main/packages/eslint-plugin-mobx#mobxexhaustive-make-observable)
, проверяющее, что все компоненты обёрнуты в observer.

### Потерян this

❌ У класса есть observer, а компонент не перерисовывается из-за потери this:

```typescript jsx
class Store {
  count = 0

  constructor() {
    makeAutoObservable(this)
  }

  increment() {
    this.count++
  }
}

const store = new Store()

const Counter = observer(() => {
  // Изменилась запись обработчик клика 👇
  return <button onClick={store.increment}>Clicked times: {store.count}</button>
})
```

This потерялся из-за передачи метода стора в обработчик клика напрямую. Нередко у разработчиков возникают сложности с this, однако если знать простое правило, то проблем с this не возникнет.
Так как в книге рассматриваются только классы, то для них достаточно знать, что **this у методов теряется, если вызывать
метод отдельно от объекта**. Разберём это правило: **вызов метода** осуществляется через парные скобки, а **отдельно от объекта** значит, что при вызове мы уже не пишем объект.
Таблица с примерами:

| Код                                                                                   | Объяснение                                                                                  | Результат       |
|---------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|-----------------|
| ``` <button onClick={() => store.increment()}> ```                                    | Вызов функции с помощью скобок не отделён от объекта                                        | Нет потери this |
| ```const increment = store.increment``` <br/><br/> ```<button onClick={increment}>``` | Вызов функции отделён от объекта                                                            | Потерян this    |
| ```<button onClick={store.increment}>```                                              | Вызов функции отделён от объекта. Это то же самое, что и пример выше, но в укороченной форме | Потерян this    |
| ```const { increment } = store``` <br/><br/> ```<button onClick={increment}>```       | Вызов функции отделён от объекта                                                            | Потерян this    |

Однако есть способы исправить все эти примеры.

✅  **Опция autoBind в makeAutoObservable**:

```typescript jsx
class Store {
  count = 0

  constructor() {
    // Добавили 1 раз опцию на весь класс 👇
    makeAutoObservable(this, {}, { autoBind: true })
  }

  increment() {
    this.count++
  }
}

const store = new Store()

const Counter = observer(() => {
  // Можем вызывать метод отдельно от объекта или использовать деструктуризацию
  return <button onClick={store.increment}>Clicked times: {store.count}</button>
})
```

Эта опция автоматически привязывает this для всех методов класса.

✅ **Методы-стрелочные функции**:

```typescript jsx
class Store {
  count = 0

  constructor() {
    makeAutoObservable(this)
  }

  // Для каждого метода используем стрелочные функции
  increment = () => {
    this.count++
  }
}

const store = new Store()

const Counter = observer(() => {
  return <button onClick={store.increment}>Clicked times: {store.count}</button>
})
```

Плюсом этого подхода является независимость от Mobx, недостатком - необходимость использовать его во всех методах. Если вас интересует более академическое объяснение того как работает this, то с ним можно ознакомиться на
сайте [learn.javascript.ru](https://learn.javascript.ru/object-methods).

### Вложенные observable и сторонние не observer компоненты

Проблема часто возникает при использовании сторонних UI-китов. Например, вы используете
компонент [Table](https://ant.design/components/table/) из UI-кита Ant Design. Этот компонент в качестве prop принимает
массив объектов. Так как компонент Table не является observer, то он не умеет подписываться на изменения объектов внутри
массива, например `store.users.isActive = true`. Компонент Table не такой умный как observer-компоненты, поэтому ему для
перерисовки нужно, чтобы ссылка на массив с объектами изменилась. Похожая проблема может встречаться у
компонента [FlatList](https://reactnative.dev/docs/flatlist) из React Native.

❌ Сторонний компонент не перерисовывается, так как он не observer, а потому не отслеживает изменения вложенных полей:

```typescript jsx
type User = {
  id: number;
  isActive: boolean;
}

class Store {
  users: User[] = [
    { id: 1, isActive: false },
    { id: 2, isActive: false },
  ]

  constructor() {
    makeAutoObservable(this, {}, { autoBind: true })
  }

  markActive(user: User) {
    user.isActive = true
  }
}

const store = new Store()

const Component = observer(() => {
  return <VendorTable
    data={store.users}
    columns={[
      {
        label: 'Mark active',
        render: (user) => {
          return <Switcher onClick={() => store.markActive(user.id)} />
        }
      }
    ]}
  />
})
```

Выходов есть несколько.

✅ Используем toJS для конвертации observable значений в чистый JS объект или массив, что даёт новую ссылку на каждый
рендер

```typescript jsx
import { toJS } from 'mobx'

type User = {
  // ...
}

class Store {
  // ...
}

const store = new Store()

const Component = observer(() => {
  return <VendorTable
    // Добавлен toJS 👇
    data={toJS(store.users)}
    columns={[
      // ...
    ]}
  />
})
```

✅ Пишем собственный компонент Table, обёрнутый в observer

```typescript jsx
import { toJS } from 'mobx'

type User = {
  // ...
}

class Store {
  // ...
}

const store = new Store()

const Component = observer(() => {
  return <MyTable
    data={store.users}
    columns={[
      // ...
    ]}
  />
})
```

Написание собственного компонента таблицы может окупить себя на длинной дистанции в сложном проекте. Таблицы это комплексные
компоненты, часто требующие самостоятельной разработки. Вот лишь малый список того, что вас могут попросить сделать:
массовое редактирование строк, иерархические таблицы, возможность сворачивать/разворачивать строки по высоте,
синхронизация с URL, бесконечный скроллинг или полностью кастомный дизайн, что может быть затруднительно при
использовании UI-китов вроде Ant Design. В таком случае можно написать собственный компонент Table с использованием
observer для поддержки реактивности.

### Render-props

Эта проблема может возникнуть с компонентами, у которых props являются функциями. В документации React этому дано
название [render prop](https://reactjs.org/docs/render-props.html). После появления хуков такие ситуации встречаются
всё реже, но давайте рассмотрим одну из них на примере [React Final Form](https://final-form.org/react):

❌ Компонент не перерисовывается при изменении `store.languages`:
```typescript jsx
import { Form } from 'react-final-form'

const MyForm = observer(() => {
  return (
    <Form
      render={({ handleSubmit }) => (
        <form onSubmit={handleSubmit}>
          <Dropdown
            label={'language'}
            options={store.languages}
            value={store.language}
            onChange={store.changeLanguage}
          />
        </form>)}
    />
  )
})
```

В этом примере `MyForm` это observer, но функция, передающаяся как проп `render`, им не является, а значит
она не может выполнить подписку на observable значения. Как следствие, даже если поле `store.languages` обновится после
загрузки данных с сервера, то компонент всё равно не перерисуется. Есть несколько вариантов решений.

✅ Используем Observer из `mobx-react-lite`/`mobx-react`

```typescript jsx
import { Form } from 'react-final-form'
import { Observer } from 'mobx-react-lite'

const MyForm = observer(() => {
  return (
    <Form
      render={({ handleSubmit }) => (
        <Observer>
          {() => <form onSubmit={handleSubmit}>
            <Dropdown
              label={'language'}
              options={store.languages}
              value={store.language}
              onChange={store.changeLanguage}
            />
          </form>}
        </Observer>)}
    />
  )
})
```

Этот компонент доступен как в mobx-react-lite, так и в mobx-react (так как второй [экспортирует](https://github.com/mobxjs/mobx/blob/b82c7f3229439a6a1f0d35ebb559dc6b0fd0bec7/packages/mobx-react/src/index.ts#L7+L17) первый).

✅ Отказываемся от render props в пользу хуков:

```typescript jsx
import { Observer } from 'mobx-react-lite'
import { useForm } from 'react-final-form-hooks'

const MyForm = observer(() => {
  const { handleSubmit } = useForm();
  
  return (
    <form onSubmit={handleSubmit}>
      <Dropdown
        label={'language'}
        options={store.languages}
        value={store.language}
        onChange={store.changeLanguage}
      />
    </form>
  )
})
```

Мы рассмотрели основные сценарии потери реактивности. Если освоить инструмент, который вы используете, то проблем, в общем случае, возникать не должно, так как все вышеперечисленные проблемы (кроме this) об одном и том же - отсутствие подписки.
