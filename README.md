# Frontend Clean Architecture

Пример приложения, собранного по трёхслойной архитектуре.

- [Слайды и полезные ссылки](https://bespoyasov.ru/talks/podlodka-conf-clean-architecture/);
- [Пример работы приложения](https://bespoyasov.ru/showcase/frontend-clean-architecture/);
- [Запись доклада](https://youtu.be/h4WQRqNjmX0).

## Что учесть

В коде есть несколько компромиссов, о которых я рассказывал в докладе на Frontend Crew. 

### Shared Kernel

Shared Kernel — это тот код и те данные, от которых могут зависеть любые модули, но зависимость от которых не повышает зацепление.

В этом приложении shared kernel включает в себя аннотации типов, которые могут быть доступны где угодно. Такие типы собраны в `shared-kernel.d.ts`.

### «Зависимость» в домене

В функции `createOrder` используется «библиотечная» функция `currentDatetime` для указания даты создания заказа. Это не совсем корректно, потому что домен не должен ни от чего зависеть.

По-хорошему, реализация типа `Order` должна быть классом, аргументами конструктора которого были бы все необходимые данные, включая дату. А создание этого класса находилось бы в прикладном слое в `orderProducts`:

```ts
async function orderProducts(user: User, products: Product[]) {
  const datetime = currentDatetime();
  const order = new Order(user, products, datetime);

  // ...
}
```

### Предоставление и тестируемость юз-кейса

Сама [функция создания заказа `orderProduct`](https://github.com/bespoyasov/frontend-clean-architecture/blob/master/src/application/orderProducts.ts#L24) не зависит от фреймворка и может быть использована и протестирована в отрыве от Реакта. Хук-обёртка используется лишь для предоставления юз-кейса компонентам и внедрения сервисов в сам юз-кейс.

В каноническом исполнении функция юз-кейса была бы вынесена за пределы хука, а сервисы были бы переданы юз-кейсу через последний аргумент или с помощью DI:

```ts
type Dependencies = {
  notifier?: NotificationService;
  payment?: PaymentService;
  orderStorage?: OrderStorageService;
}

async function orderProducts(user: User, cart: Cart, dependencies: Dependencies = defaultDependencies) {
  const {notifier, payment, orderStorage} = dependencies;
  
  // ...
}
```

Хук в этом случае превратился бы в адаптер:

```ts
function useOrderProducts() {
  const notifier = useNotifier();
  const payment = usePayment();
  const orderStorage = useOrdersStorage();
  
  return (user: User, cart: Cart) => orderProducts(user, cart, {
    notifier, payment, orderStorage,
  })
}
```

В докладе я обращаю внимание этот момент и поясняю, как будет сделать правильно с точки зрения чистоты подхода. В исходниках же я посчитал, что это необязательно, так как отвлекало бы от сути. Также это — один из «срезанных углов», о которых я упоминаю в начале доклада.

### Кустарный DI

В прикладном слое мы «внедряем» сервисы руками:

```ts
export function useAuthenticate() {
  const storage: UserStorageService = useUserStorage();
  const auth: AuthenticationService = useAuth();

  // ...
}
```

По-хорошему, это должно быть автоматизировано и сделано через внедрение зависимостей. В случае с Реактом и хуками мы, в принципе, можем использовать их, как «контейнер», который возвращает реализацию указанного интерфейса.

В конкретно этом приложении настраивать DI особо смысла не было, потому что это бы отвлекало от сути архитектуры.
