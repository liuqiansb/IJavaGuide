##### 基本使用

```javascript
function greeter(person: string) {
    return "Hello, " + person;
}

let user = "Jane User";

document.body.innerHTML = greeter(user);
```

##### 接口

```typescript
interface Person{
	firstName:string;
    lastName:string;
}

function greeter(person:Person){
    return "Hello,"+person.firstName+""+person.lastName;
}
let user = {firstName:"Jane",lastName:"User"};
document.body.innerHTML = greeter(user)

注意：基于接口传参并不需要implements这样的关键字，只要传入的这个对象它有接口定义的那两个属性便可以编译通过
```

##### 类

```typescript
class Student {
    fullName: string;
    // 注意在构造方法上使用public相当于定义了类的同名属性
    constructor(public firstName, public middleInitial, public lastName) {
        this.fullName = firstName + " " + middleInitial + " " + lastName;
    }
}

interface Person {
    firstName: string;
    lastName: string;
}

function greeter(person : Person) {
    return "Hello, " + person.firstName + " " + person.lastName;
}

let user = new Student("Jane", "M.", "User");

document.body.innerHTML = greeter(user);
```

