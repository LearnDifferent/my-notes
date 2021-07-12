[toc]

# TypeScript 基础

`npm install -g typescript`

可以使用 `tsc [文件名].ts` 来将 TS 文件编译为 JS 文件。

实时编译：`tsc [文件名].ts -w` 

基本语法：

```typescript
let a: number;
a = 45;

let b: string;
b = 'only string';

let c = false;
c = true;

let d: "male" | "female";
d = "male";
d = "female";
// d = "无法赋值为其他"

let e: string | boolean;
e = '可以是 string 类型，也可以是 boolean';
e = true;


let f: any;
f = '变为了 JavaScript 的方式，但是不建议用';

let g: unknown;
g = true;
g = 1;
g = '建议用 unknown 取代 any';

// h 的类型为 number
let h: number;
// 如果是 any 类型的 f，可以直接赋值，不会报错
// 但是 f 其实定义了 string，所以会出现问题
h = f;
// 而 g 就能检测出问题：
// h = g;

// 如果是使用 unknown
let i: unknown;
// 将其赋值为 number 类型的 11
i = 11;
// 就算新变量的类型也是 number
let j: number;
// 因为 unknown 是类型安全的 any，所以不能直接赋值：
// j = i;
if (typeof i === 'number') {
    // 而是要先判断类型后，才能赋值
    j = i;
}
// 或者使用类型断言
j = i as number;
// 类型断言除了 as，还能这样：
j = <number>i;

// 符号为 {} 时，定义对象，语法为 {属性名: 属性值}
let ppl0: { name: string, age: number; };
ppl0 = {name: 'john', age: 11};
// 如果属性名后面加上问号，表示不一定要指定该属性
let ppl1: { name?: string; };
ppl1 = {};
let ppl2: { name: string, age?: number; };
ppl2 = {name: "john"};
// 如果只想定义一个属性，其他属性随意添加（propName 这一个可以随意写，相当于局部变量名）：
let ppl3: { name: string, [propName: string]: unknown; };
ppl3 = {name: "john", age: 13, male: true};
ppl3 = {name: "john", age: 13};

// 定义 function 的结构：
let fun1: (a: number, b: number) => number;
// 所以该函数只能：局部变量 a 为 number，b 为 number，返回值为 number
fun1 = function (n1, n2) {
    return 1;
};

// 定义数组的同时，指定能存储哪些类型的数据
let strArr: string[];
strArr = ['只能放入字符串', '不能放入其他类型'];
// 还能用这样的语法：
let numArr: Array<number>;
numArr = [1, 2, 3];

// 如果要定义固定长度和类型的数组，也就是 元组：
let fixArr: [string, string, string];
fixArr = ["只有三个", "不能多也不能少", "而且类型必须对应"];
let fixStrNum: [string, number];
fixStrNum = ["只要类型和个数对应即可", 2];

// 可以定义枚举
enum Gender {
    Male, Female
}

// 枚举可以这样使用和判断：
let ppl4: { name: string, gender: Gender; };
ppl4 = {name: "john", gender: Gender.Male};
console.log(ppl4.gender === Gender.Male);

// 可以给类型起别名
type myType = 1 | 2;
let m: myType;
m = 1;
// m = 3;

// 局部变量 a 为 number，b 为 number，返回值为 number
function sum(a: number, b: number): number {
    return a + b;
}

// 局部变量 a 为 number，b 为 number，没有返回值
function logSum(a: number, b: number): void {
    // 方法定义的 void 也可以不写
    console.log(a + b);
}

// 返回值为 never，用于抛出异常
function errorSum(): never {
    throw new Error("出现异常")
}

// 局部变量 a 为 number，b 为 number，返回值可以是 string 或 number
function moreThanZero(a: number, b: number): number | string {
    if (a + b > 0) {
        return a + b;
    } else {
        return "小于或等于 0";
    }
}

```

类（参考 Java 即可）：

```typescript
class Person {
    // 定义属性
    name: string;
    // 属性可以赋值，可以定义为私有属性
    private _age: number;
    // 可以定义静态属性/类属性，也可以赋值
    static isCreature: boolean = true;
    // 可以让属性只读
    readonly iq: number;

    // 定义构造函数，创建时赋值
    constructor(name: string, age: number, iq: number) {
        this.name = name;
        this._age = age;
        this.iq = iq;
    }

    // getter
    get age(): number {
        return this._age;
    }

    // setter
    set age(value: number) {
        this._age = value;
    }

    // 定义方法
    getName(): string {
        return this.name;
    }

    // 打印当前对象
    getPerson() {
        console.log(this);
    }

    // 可以定义静态方法
    static eat(food: string): void {
        console.log("eating " + food);
    }
}

let john = new Person("john", 15, 200);
john.name = "sally";
console.log(john.age);
console.log(Person.isCreature);
john.getPerson();
```

接口：

```typescript
interface myType {
    name: string;
    readonly age: number;
}
// 可以用接口来当作类型
const john: myType = {
    name: "John",
    age: 11
}

interface printName {
    name: string;
    printOne(): void;
}

class John implements printName {
    name: string;

    constructor(name: string) {
        this.name = name;
    }

    printOne(): void {
        console.log(this.name);
    }
}
```

范型：

```typescript
// 范型相当于用变量来表示类型
function returnOrigin<T>(ori: T): T {
    return ori;
}

returnOrigin("如果不知道类型，就这样定义，会自动判断类型");
returnOrigin<string>('还可以自己指定类型，返回的时候一定就是字符串')

function getFirst<T, K>(a: T, b: K): T {
    // 传入两个参数，只返回第一个参数，并识别该参数的类型
    return a;
}

getFirst("String", 1);
getFirst<boolean, string>(true, "还可以手动指定");

interface Length {
    // 定义接口，必须有属性 length
    length: number;
}

function getLen<T extends Length>(a: T): void {
    // T extends Length 表示 T 可以是任意类型，但是继承了 Length 接口
    // 因为 Length 接口内有 length 属性
    // ，所以传入类型为 T 的 a 变量肯定也有 length 属性
    console.log(a.length);
}

let johnHeight: { name: string; length: number }
johnHeight = {name: "john", length: 13}
// getLen 方法会打印 johnHeight.length
getLen(johnHeight);

// 类上面也可以定义范型
class Something<T> {
    private _nameOrAge: T;

    constructor(nameOrAge: T) {
        this._nameOrAge = nameOrAge;
    }

    get nameOrAge(): T {
        return this._nameOrAge;
    }
}

let something = new Something("此时需要 Name");
console.log(something.nameOrAge);
let onlyAgeIfNumber = new Something<number>(11);
console.log("Age 为：" + onlyAgeIfNumber.nameOrAge);
```

