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
