# Filter与type predicate

我想你肯定遇到过这样的问题。

```typescript
let room = Game.rooms['W1N1']

// 使用find里的filter回调
let towers = room.find(FIND_MY_STRUCTURES, { 
                  filter: (s) => s.structureType === STRUCTURE_TOWER 
})

towers[0].attack()  //错误! 类型“AnyOwnedStructure”上不存在属性“attack”。
                    //     类型“StructureController”上不存在属性“attack”。ts(2339)

// 使用数组的filter
let towers2: StructureTower[] = room.find(FIND_MY_STRUCTURES)
                                    .filter(s => s.structureType === STRUCTURE_TOWER)
// 错误! 不能将类型“AnyOwnedStructure[]”分配给类型“StructureTower[]”。
// 不能将类型“AnyOwnedStructure”分配给类型“StructureTower”
```

为什么在这种情况下我们的filter没有将过滤后的数组的类型收束为StructureTower\[]呢？

查看一下filter方法的类型声明，也许可以解答你的疑惑。

```typescript
/**
 * Returns the elements of an array that meet the condition specified in a callback function.
 * @param predicate A function that accepts up to three arguments. The filter method calls the predicate function one time for each element in the array.
 * @param thisArg An object to which the this keyword can refer in the predicate function. If thisArg is omitted, undefined is used as the this value.
 */
filter<S extends T>(predicate: (value: T, index: number, array: T[]) => value is S, thisArg?: any): S[];

/**
 * Returns the elements of an array that meet the condition specified in a callback function.
 * @param predicate A function that accepts up to three arguments. The filter method calls the predicate function one time for each element in the array.
 * @param thisArg An object to which the this keyword can refer in the predicate function. If thisArg is omitted, undefined is used as the this value.
 */
filter(predicate: (value: T, index: number, array: T[]) => unknown, thisArg?: any): T[];
```

可以看到TS官方为数组的filter方法声明了两种类型, 我们传入的predicate函数没有上面的那些奇怪的修饰`=>value is S`，因此命中了下面的类型。而在下面的filter中，过滤后的数组类型不会发生改变，所以`room.find`后得到的`AnyOwnedStructure[]`在经历filter后，仍是`AnyOwnedStructure[]`。

（`AnyOwnedStructure`是所有`FIND_MY_STRUCTURE`可以找到的建筑类型的联合类型）

当然，filter不能过滤类型岂不是太弱了？因此TS官方提供了上面的filter类型。你应该可以轻松的照着写出一个专用于Tower的filter。

```typescript
// 没有报错!
let towers3: StructureTower[] = room
  .find(FIND_MY_STRUCTURES)
  .filter((s): s is StructureTower => s.structureType === STRUCTURE_TOWER)
```

这里的 `:[参数名] is [类型名]` 被ts官方称作type predicate。

## 改进

然而，如果每次都要写一个`s is StructureXXX`，相当于要输两遍目标类型，这也太麻烦了吧？

甚至，如果你像我一样懒得给每个变量标注类型，同时手滑输错了 is 后面的类型，会发生这样荒谬的事

<pre class="language-typescript"><code class="lang-typescript"><strong>// towers3 的类型变成了 StructureSpawn[] !!!
</strong><strong>let towers3 = room
</strong>  .find(FIND_MY_STRUCTURES)
  .filter((s): s is StructureSpawn => s.structureType === STRUCTURE_TOWER)
</code></pre>

你可能会想，这和我手动`as Structure[]` 有什么大区别？

我们当然不能容忍这样的事情发生。Typescript应当是给我们带来便利的，而不应增添我们的烦恼。

我们知道，filter的参数应该是一个函数。我们也知道，一个函数可以生成另一个函数，所以，我们可以实现一个工具（工厂）函数，自动返回一个带有type predicate和过滤条件的函数。

提示：你可以在typed-screeps的类型声明文件中搜索

`ConcreteStructure<T extends StructureConstant>`&#x20;

这是一个高阶工具类型，类型参数为建筑类型常量，返回类型为对应的建筑类型。如

```typescript
ConcreteStructure<STRUCTURE_SPAWN>    =>    StructureSpawn
```





























答案

```typescript
export function isStructureType<T extends StructureConstant>(structureType: T) {
  return (s: AnyStructure): s is ConcreteStructure<T> => {
    return s.structureType === structureType;
  }
}
```

使用的时候，只需要像这样传给filter即可

```typescript
// 类型自动被推断为StructureLink[]了!
let links = room.find(FIND_STRUCTURE)
                .filter(isStructureType(STRUCTURE_LINK))
```

题外话：如果想过滤出多种建筑类型该怎么做？

我不知是否应该写在这里，因为这涉及到了别的内容。如果你想知道答案可以看[这里](https://github.com/DiamondMofeng/ScreepsPlaying/blob/main/src/utils/typer.ts#L22)
