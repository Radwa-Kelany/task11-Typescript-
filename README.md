# Task11-Typescript-code
## Implementing a Retry Mechanism for Asynchronous Functions
### General concept
This Mechanism aims for providing some retrial times for async function when failed,
its useful in case of DB connection and HTTP requests.
### Flowchart
<img width="360" height="440" alt="Retry request flowchart" src="https://github.com/user-attachments/assets/28bae653-f94d-48e8-9c01-aeb4c178cf59" />

### Code 
```ts
class Retry {
  attempts_count: number;
  waiting_time: number;
  constructor(count: number, time: number) {
    this.attempts_count = count;
    this.waiting_time = time;
  }
  async checkRetry<T>(DBConfig: T, f: (config: T) => boolean) {
    let connect = f(DBConfig);
    for (let i = 1; i <= this.attempts_count!; i++) {
      if (connect) {
        break;
      } else {
        console.log(`attempt ${i} of ${this.attempts_count}`);
        await new Promise((resolve) => setTimeout(resolve, this.waiting_time));
      }
    }
    return connect;
  }
}

function checkDBConnection(test: string): boolean {
  return test === 'SQL';
}

let dbConnection = new Retry(3, 3000);

dbConnection.checkRetry<string>('Postgres', checkDBConnection);

```
## Building a Promise-Based Cache with Expiration
### General concept
This Mechanism aims for getting data from cache, and if it's expire or not in cache ---> we send a promise request to DB for retrieving data.
At the same time we prevent duplication of request for same data. When first user send a request to DB, we save a promise for the request in cache, that returns to next following users who ask for same data.
Then after the promise resolved, the data sent to all users. This promise based cache helps in preventing request duplication and save server resources.
### Flowchart
<img width="710" height="720" alt="promise cache flowchart" src="https://github.com/user-attachments/assets/baf7805e-876e-406f-b92b-db1a8d44cc39" />


### Code 
```ts
// Simulate DB
type entityDB = Array<{
  data: {
    id: number;
    name: string;
  };
}>;
let UsersDB: entityDB = [
  {
    data: {
      id: 1,
      name: 'Sid',
    },
  },
  {
    data: {
      id: 2,
      name: 'John',
    },
  },
  {
    data: {
      id: 3,
      name: 'Kelly',
    },
  },
];
// Simulate Cache
type entityCache<T> = Array<{
  data: {
    id: number;
    name: string;
  };
  ttl: number;
  promise?: Promise<T>;
}>;

let UsersCache: entityCache<any> = [
  {
    data: {
      id: 1,
      name: 'Lim',
    },
    ttl: new Date('02-11-2026').getTime(),
  },
  {
    data: {
      id: 2,
      name: 'John',
    },
    ttl: new Date('02-08-2026').getTime(),
  },
];

```
```ts
// fetcher.ts
export const fetchUserData = async <T = any>(
  userId: number,
  ttl: number,
): Promise<T> => {
  let user = UsersCache.find((use) => use.data.id == userId);
  try {
    // Fetch user from DB
    const userDB = UsersDB.find((use) => use.data.id == userId);
    if (userDB) {
      let now = Date.now();
      let newUser = {
        data: userDB.data,
        ttl: now + ttl,
      };
      let userCache = UsersCache.find((use) => use.data.id == userDB.data.id);
      if (userCache) {
        userCache = newUser;
      } else {
        UsersCache.push(newUser);
      }
      await new Promise((resolve) => setTimeout(resolve, 3000)); // Simulate delay
      user = newUser;
    }
    return user as T;
  } catch (e) {
    return e as T;
  }
};
```
```ts

// CachePromise.ts
// import fetchUserData from 'fetcher.ts'
export class CachePromise<T = any> {
  defaultTTL: number;
  entity: entityCache<T>[0] = {} as entityCache<T>[0];
  constructor(TTL: number) {
    this.defaultTTL = TTL;
  }
  getCache(entityId: number) {
    const found = UsersCache.find((use) => use.data.id === entityId);
    if (found) {
      this.entity = found;
    } else {
      this.entity = {} as entityCache<T>[0];
    }
  }
  async getOrCreateCache(
    userId: number,
    fetcher: () => Promise<T>,
  ): Promise<T> {
    let now = Date.now();
    // get user from Cache
    this.getCache(userId);
    console.log(this.entity);
    // Check if user is found and not expired
    if (this.entity && this.entity.ttl > now) {
      return this.entity as T;
    }
    //  Check if user is in promise = running call request
    if (this.entity?.promise) {
      return this.entity.promise as T;
    }

    // Create promise to get user from DB and update it in cache

    let promise = fetcher()
      .then((result) => console.log(result))
      .catch((e) => console.log(e));

    // return pending promise while promise is resolving
    if (this.entity) {
      this.entity.data = { id: userId, name: null as any };
      this.entity.ttl = now + this.defaultTTL;
      this.entity.promise = fetcher();
    } else {
      UsersCache.push({
        data: { id: userId, name: null as any },
        ttl: now + this.defaultTTL,
        promise: fetcher(),
      });
    }
    return (await promise) as T;
  }
}
```
```ts
// app.ts
// import cachePromise from cachePromise.ts
let cache = new CachePromise<any>(5 * 60 * 1000);
let user = cache.getOrCreateCache(2, () => fetchUserData(2, cache.defaultTTL));
```
## Implementing a Rate-Limited API With Typescript
### General concept
This Mechanism aims throttling excess requests to API endpoints to prevent server fall down or protect it from DoS attacks.
For example: we can only allow 1000 requests /minute. If the number of requests exceed Max-limit, we throttling them and send waiting response.
### Flowchart
<img width="350" height="510" alt="rate limit flowchart" src="https://github.com/user-attachments/assets/df919b2d-2bca-4d8e-943f-bc78ba29bf90" />


### Code 
```ts
// requestRate.ts
export class RequestRate {
  max_requests_count: number;
  allowedTime: number;
  startTime: number;
  current_request_count: number = 10;

  constructor(count: number, time: number) {
    this.max_requests_count = count;
    this.allowedTime = time;
    this.startTime = Date.now() + this.allowedTime;
  }

  checkStartTime() {
    let now = Date.now();
    if (now - this.startTime! > this.allowedTime!) {
      this.current_request_count = 0;
      this.startTime = now + this.allowedTime!;
    }
  }
  allowRequest(): boolean {
    this.checkStartTime();
    if (this.current_request_count > this.max_requests_count!) {
      return false;
    } else {
      this.current_request_count++;
      return true;
    }
  }
  middleWare() {
    return this.allowRequest();
  }
}
```
```ts
// middleWare.ts
// import RequestRate from requestRate.ts
let orderRateLimit = new RequestRate(9, 1000);
export const requestRateMiddleWare = function (
  req: Request,
  res: Response,
  next: Function,
): void {
  if (orderRateLimit.middleWare()) {
    console.log('yes allowed');
    next();
  } else throw new Error('Exceed Rate limit');
};
```
```ts
// app.ts
// import requestRateMiddleWare

app.post('/api/v1/order/add', requestRateMiddleWare(), (req, res) => {
  res.json({ msg: 'successful order submitting' });
});

```
