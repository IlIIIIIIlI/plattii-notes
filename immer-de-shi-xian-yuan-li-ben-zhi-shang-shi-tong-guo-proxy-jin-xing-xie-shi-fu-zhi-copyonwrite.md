# Immer 的实现原理本质上是通过 Proxy 进行写时复制（Copy-on-Write）

```typescript
/**
 * Draft 类型：移除对象所有属性的 readonly 修饰符
 * 这样我们就可以在 recipe 函数中"看似"直接修改对象属性
 * 例如：draft.user.name = 'new name' 在类型检查时不会报错
 */
type Draft<T> = {
  -readonly [P in keyof T]: T[P];
};

/**
 * Immer 风格的不可变更新函数
 * 
 * 核心思想：
 * 1. 创建原对象的 Proxy 代理，拦截所有读写操作
 * 2. 允许用户通过"可变"语法修改 draft 对象
 * 3. 在背后记录所有修改，最终生成新的不可变对象
 * 4. 如果没有修改，直接返回原对象（引用相等优化）
 * 
 * @param base 原始对象，不会被修改
 * @param recipe 修改函数，接收 draft 对象进行修改
 * @returns 新的对象（如有修改）或原对象（如无修改）
 */
function produce<T>(base: T, recipe: (draft: Draft<T>) => void): T {
  /**
   * WeakMap 用于存储每个对象的代理信息
   * key: 原始对象
   * value: { proxy: 代理对象, modified: 修改记录 }
   * 
   * 使用 WeakMap 的好处：
   * 1. 不会阻止垃圾回收
   * 2. 可以用对象作为 key
   * 3. 避免重复创建相同对象的代理
   */
  const modifiedMap = new WeakMap();
  
  /**
   * 全局标记，用于快速判断是否有任何修改
   * 如果为 false，可以直接返回原对象，避免不必要的对象创建
   */
  let hasAnyModifications = false;
  
  /**
   * 核心函数：为给定对象创建 Proxy 代理
   * 
   * 这个函数会递归地为嵌套对象创建代理，实现深度拦截
   * 
   * @param target 需要代理的目标对象
   * @returns 代理对象
   */
  const createProxy = (target: any): any => {
    /**
     * 缓存优化：如果已经为这个对象创建过代理，直接返回
     * 避免重复创建，提高性能
     */
    if (modifiedMap.has(target)) {
      return modifiedMap.get(target).proxy;
    }
    
    /**
     * 当前对象层级的修改记录
     * key: 属性名
     * value: 修改后的值
     */
    const modified: Record<string, any> = {};
    
    /**
     * 创建 Proxy 代理对象
     * Proxy 是 ES6 提供的元编程特性，可以拦截对象的各种操作
     */
    const proxy = new Proxy(target, {
      /**
       * get trap：拦截属性读取操作
       * 当执行 draft.user.name 时会触发这个函数
       * 
       * @param obj 目标对象
       * @param prop 访问的属性名
       * @returns 属性值或子代理对象
       */
      get(obj, prop: string) {
        /**
         * 优先返回修改后的值
         * 如果这个属性已经被修改过，直接返回新值
         * 这样就实现了"写后读"的一致性
         */
        if (prop in modified) {
          return modified[prop];
        }
        
        // 获取原始属性值
        const value = obj[prop];
        
        /**
         * 递归代理：如果属性值是对象，为其创建子代理
         * 
         * 条件判断：
         * 1. typeof value === 'object' - 是对象类型
         * 2. value !== null - 不是 null（null 的 typeof 也是 'object'）
         * 3. !Array.isArray(value) - 不是数组（简化处理，实际 Immer 也支持数组）
         * 
         * 这样当用户访问 draft.user 时，返回的是 user 对象的代理
         * 然后访问 draft.user.name 时，又会在 user 代理上触发 get trap
         */
        if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
          return createProxy(value);
        }
        
        // 基本类型直接返回
        return value;
      },
      
      /**
       * set trap：拦截属性写入操作
       * 当执行 draft.user.name = 'new name' 时会触发这个函数
       * 
       * @param obj 目标对象
       * @param prop 设置的属性名
       * @param value 设置的新值
       * @returns 必须返回 true 表示设置成功
       */
      set(obj, prop: string, value: any) {
        /**
         * 性能优化：只有当值真的改变时才记录修改
         * 避免无效的对象创建，例如：
         * draft.user.name = draft.user.name; // 设置相同值，不应该触发修改
         */
        if (obj[prop] !== value) {
          // 在当前层级记录修改
          modified[prop] = value;
          
          // 设置全局修改标记
          hasAnyModifications = true;
        }
        
        /**
         * 返回 true 表示属性设置成功
         * 这是 Proxy set trap 的要求
         */
        return true;
      }
    });
    
    /**
     * 缓存代理对象和修改记录
     * 这样下次访问同一个对象时可以直接返回已存在的代理
     */
    modifiedMap.set(target, { proxy, modified });
    
    return proxy;
  };
  
  // 为根对象创建代理
  const draft = createProxy(base);
  
  /**
   * 执行用户的修改函数
   * 用户在这个函数中的所有修改操作都会被 Proxy 拦截和记录
   * 例如：draft.user.name = 'new name' 会触发多层 get 和最终的 set
   */
  recipe(draft);
  
  /**
   * 性能优化：如果没有任何修改，直接返回原对象
   * 这样可以保持引用相等性，对 React 等框架的性能优化很重要
   * 例如：React.memo 可以通过 === 比较来避免不必要的重渲染
   */
  if (!hasAnyModifications) {
    return base;
  }
  
  /**
   * 生成最终结果：深拷贝代理对象
   * 
   * 为什么用 JSON.parse(JSON.stringify)：
   * 1. 简单有效的深拷贝方法
   * 2. 当 JSON.stringify 遍历对象时会触发所有 get trap
   * 3. 这时 modified 中的值会被正确返回，实现修改的合并
   * 4. 最终得到一个全新的对象，包含所有修改
   * 
   * 注意：这种方法有局限性（不支持函数、Date、RegExp 等），
   * 实际的 Immer 使用更复杂的拷贝算法
   */
  const result = JSON.parse(JSON.stringify(draft));
  return result;
}

/**
 * 测试数据：模拟一个典型的应用状态
 * 包含多层嵌套结构，用于测试不同场景
 */
const state = {
  user: {
    name: '张三',
    age: 25,
    profile: {
      city: '北京',
      hobbies: ['reading', 'coding']
    }
  },
  count: 0
};

/**
 * 测试1: 修改深层嵌套属性
 * 
 * 执行过程分析：
 * 1. draft.user -> 触发 get trap，返回 user 对象的代理
 * 2. draft.user.name = '李四' -> 在 user 代理上触发 set trap
 * 3. 在 user 层级的 modified 对象中记录：{ name: '李四' }
 * 4. hasAnyModifications 被设为 true
 * 5. 最终序列化时，user.name 会从 modified 中返回新值
 */
console.log('=== 测试1: 只修改嵌套属性 ===');
const newState1 = produce(state, draft => {
  draft.user.name = '李四';  // 修改深层属性
});
console.log('原始对象中的 name:', state.user.name);       // 张三 (原对象未变)
console.log('新对象中的 name:', newState1.user.name);     // 李四 (新对象已变)
console.log('两个对象是否相同:', state === newState1);    // false (创建了新对象)
console.log('user 对象是否相同:', state.user === newState1.user); // false (嵌套对象也是新的)

/**
 * 测试2: 只访问不修改的优化场景
 * 
 * 执行过程分析：
 * 1. draft.user.name -> 触发多层 get trap，但没有 set trap
 * 2. hasAnyModifications 保持为 false
 * 3. 直接返回原对象，实现引用相等优化
 * 
 * 这个优化对性能很重要，特别是在 React 等框架中
 */
console.log('\n=== 测试2: 无修改时返回原对象 ===');
const newState2 = produce(state, draft => {
  // 只访问属性，不做任何修改
  const name = draft.user.name;
  const age = draft.user.age;
  // 这些操作只会触发 get trap，不会触发 set trap
});
console.log('是否返回原对象:', state === newState2); // true (性能优化)

/**
 * 测试3: 修改多个层级和多个属性
 * 
 * 执行过程分析：
 * 1. draft.user.profile.city = '上海' -> 在 profile 层级记录修改
 * 2. draft.count = 1 -> 在根层级记录修改
 * 3. 不同层级的修改都会被正确跟踪和应用
 */
console.log('\n=== 测试3: 修改多层嵌套 ===');
const newState3 = produce(state, draft => {
  draft.user.profile.city = '上海';  // 修改深层嵌套属性
  draft.count = 1;                   // 修改根层级属性
});
console.log('原始城市:', state.user.profile.city);       // 北京
console.log('新的城市:', newState3.user.profile.city);   // 上海
console.log('原始计数:', state.count);                   // 0
console.log('新的计数:', newState3.count);               // 1
console.log('原始对象完全不变:', JSON.stringify(state));  // 验证原对象未被污染

/**
 * 测试4: 设置相同值的优化
 * 
 * 执行过程分析：
 * 1. draft.user.name = '张三' -> 触发 set trap
 * 2. 由于 obj[prop] !== value 检查，发现值相同
 * 3. 不记录修改，hasAnyModifications 保持 false
 * 4. 返回原对象，避免不必要的对象创建
 */
console.log('\n=== 测试4: 相同值不触发修改 ===');
const newState4 = produce(state, draft => {
  draft.user.name = '张三'; // 设置与原值相同的值
});
console.log('设置相同值是否返回原对象:', state === newState4); // true

/**
 * 测试5: 结构共享验证
 * 
 * Immer 的一个重要特性是结构共享：
 * - 被修改的对象及其父级链会创建新对象
 * - 未被修改的部分保持原始引用
 * 这样既保证了不可变性，又优化了内存使用
 */
console.log('\n=== 测试5: 结构共享验证 ===');
const newState5 = produce(state, draft => {
  draft.count = 100; // 只修改根级属性
});
console.log('根对象是否相同:', state === newState5);                    // false (根对象变了)
console.log('user 对象是否相同:', state.user === newState5.user);       // true (user 未变，共享引用)
console.log('profile 对象是否相同:', state.user.profile === newState5.user.profile); // true (更深层也共享)

/**
 * 实际应用场景示例：模拟 Redux/Zustand 状态更新
 */
console.log('\n=== 实际应用场景示例 ===');

// 模拟用户信息更新操作
const updateUserName = (state: any, newName: string) => {
  return produce(state, draft => {
    draft.user.name = newName;
  });
};

// 模拟计数器增加操作
const incrementCount = (state: any) => {
  return produce(state, draft => {
    draft.count += 1;
  });
};

let currentState = state;
console.log('初始状态:', currentState);

currentState = updateUserName(currentState, '王五');
console.log('更新姓名后:', currentState);

currentState = incrementCount(currentState);
console.log('增加计数后:', currentState);

// 验证原始状态完全未变
console.log('原始状态仍然是:', state);
```

Immer 的实现原理本质上是通过 Proxy 进行写时复制（Copy-on-Write）。

**传统不可变更新的痛点：**

假设你有这样一个状态：

```javascript
const state = {
  user: {
    profile: {
      personal: {
        name: '张三',
        age: 25
      },
      contact: {
        email: 'zhang@example.com'
      }
    }
  },
  settings: { theme: 'dark' }
}
```

如果要修改 `name`，传统做法需要：

```javascript
const newState = {
  ...state,
  user: {
    ...state.user,
    profile: {
      ...state.user.profile,
      personal: {
        ...state.user.profile.personal,
        name: '李四'
      }
    }
  }
}
```

这种展开语法在深层嵌套时变得极其冗长和易错。

**Immer 的解决方案：**

Immer 通过 Proxy 创建了一个拦截层。当你写 `draft.user.profile.personal.name = '李四'` 时，实际执行的是：

1. **访问路径拦截**：`draft.user` 触发 get trap，返回 user 的代理；`draft.user.profile` 又触发 get trap，返回 profile 的代理；以此类推。
2. **修改记录**：最终的赋值操作被 set trap 捕获，在对应层级的 `modified` 对象中记录变更。
3. **惰性拷贝**：只有被修改的路径上的对象才会被标记为"需要拷贝"，其他部分保持原始引用。

**核心机制分析：**

```javascript
// 每个代理层都维护自己的修改记录
const handler = {
  get(target, prop) {
    if (prop in modified) {
      return modified[prop]; // 返回已修改的值
    }
    
    const value = target[prop];
    if (typeof value === 'object' && value !== null) {
      return new Proxy(value, handler); // 递归代理
    }
    return value;
  },
  
  set(target, prop, value) {
    if (target[prop] !== value) {
      modified[prop] = value; // 记录修改
      hasChanges = true;
    }
    return true;
  }
}
```

**最终合成阶段：**

当 `JSON.stringify(proxy)` 执行时，会遍历整个对象树。对于每个属性访问，如果该属性被修改过，get trap 会返回 `modified` 中的新值；否则返回原始值。这样最终序列化的结果就是一个包含所有修改的新对象。

**性能优化：**

1. **引用相等优化**：如果没有任何修改，直接返回原对象
2. **结构共享**：未修改的子树在新对象中保持原始引用
3. **按需代理**：只有被访问的路径才会创建代理对象

这种设计实现了用命令式语法进行声明式更新，避免了手动展开的复杂性，同时保证了不可变性和性能。
