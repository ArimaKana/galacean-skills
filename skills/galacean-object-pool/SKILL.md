---
name: galacean-object-pool
description: 此 Skill 包含了 Galacean Engine 中对象池（Object Pool）的实现模式，用于高效管理子弹、敌人等频繁创建销毁的游戏对象
---

# Galacean 对象池（Object Pool）

## 1. 为什么需要对象池

在游戏中，频繁创建和销毁对象（如子弹、敌人）会导致：
- **性能问题**：频繁的内存分配和垃圾回收
- **卡顿**：GC 时游戏卡顿
- **内存碎片**：长期运行后内存效率降低

**对象池解决方案**：预先创建对象，重复利用，避免频繁创建销毁。

## 2. 基础对象池实现

```typescript
class ObjectPool<T extends Entity> {
  private pool: T[] = [];
  private createFn: () => T;
  private resetFn: (obj: T) => void;
  private initialSize: number;
  private initialized: boolean = false;

  constructor(
    createFn: () => T,      // 创建对象的函数
    resetFn: (obj: T) => void,  // 重置对象的函数
    initialSize: number = 10
  ) {
    this.createFn = createFn;
    this.resetFn = resetFn;
    this.initialSize = initialSize;
  }

  // 延迟初始化，避免构造函数中的循环依赖问题
  private initialize(): void {
    if (this.initialized) return;
    this.initialized = true;
    
    for (let i = 0; i < this.initialSize; i++) {
      const obj = this.createFn();
      obj.enabled = false;
      this.pool.push(obj);
    }
  }

  // 获取对象
  get(): T {
    this.initialize();
    
    let obj: T;
    if (this.pool.length > 0) {
      obj = this.pool.pop()!;
    } else {
      obj = this.createFn();
    }
    
    // 重置对象状态（设置 pool 引用、重置属性）
    this.resetFn(obj);
    
    // 启用对象
    obj.enabled = true;
    
    return obj;
  }

  // 归还对象到池中
  return(obj: T): void {
    // 禁用对象
    obj.enabled = false;
    // 移出视野（避免干扰）
    obj.transform.position.set(0, -1000, 0);
    this.pool.push(obj);
  }

  // 清空对象池
  clear(): void {
    this.pool.forEach(obj => obj.destroy());
    this.pool = [];
  }
}
```

## 3. 使用示例：子弹对象池

### 3.1 子弹脚本

```typescript
class BulletScript extends Script {
  speed: number = 15;
  direction: Vector3 = new Vector3();
  lifeTime: number = 3;
  private currentLife: number = 0;
  private pool: ObjectPool<Entity> | null = null;

  // 关键：对象池通过此方法设置引用
  setPool(pool: ObjectPool<Entity>): void {
    this.pool = pool;
  }

  onUpdate(deltaTime: number): void {
    this.currentLife += deltaTime;
    if (this.currentLife >= this.lifeTime) {
      this.recycle();
      return;
    }

    // 移动
    const moveDistance = this.speed * deltaTime;
    this.entity.transform.translate(
      this.direction.x * moveDistance,
      0,
      this.direction.z * moveDistance,
      false
    );

    // 超出边界检查
    const pos = this.entity.transform.position;
    if (pos.x < -15 || pos.x > 15 || pos.z < -15 || pos.z > 15) {
      this.recycle();
    }
  }

  // 重置对象状态（对象池调用）
  reset(): void {
    this.currentLife = 0;
    this.direction.set(0, 0, 0);
  }

  // 回收对象
  private recycle(): void {
    if (this.pool) {
      this.pool.return(this.entity);
    }
  }
}
```

### 3.2 初始化和使用

```typescript
class GameManager extends Script {
  private bulletPool: ObjectPool<Entity> | null = null;

  onAwake(): void {
    this.initPools();
  }

  private initPools(): void {
    this.bulletPool = new ObjectPool<Entity>(
      () => this.createBullet(),      // createFn
      (bullet) => this.resetBullet(bullet),  // resetFn
      30  // 初始池大小
    );
  }

  // 创建子弹（只创建基础组件）
  private createBullet(): Entity {
    const bullet = this.entity.createChild('bullet');
    
    // 视觉组件
    const meshRenderer = bullet.addComponent(MeshRenderer);
    meshRenderer.mesh = PrimitiveMesh.createSphere(this.engine, 0.3);
    const material = new BlinnPhongMaterial(this.engine);
    material.baseColor.set(1, 0.8, 0, 1);
    meshRenderer.setMaterial(material);

    // 脚本组件（不设置 pool，在 reset 时设置）
    bullet.addComponent(BulletScript);

    return bullet;
  }

  // 重置子弹（对象池获取时调用）
  private resetBullet(bullet: Entity): void {
    const script = bullet.getComponent(BulletScript);
    if (script) {
      // 关键：设置 pool 引用
      script.setPool(this.bulletPool!);
      script.reset();
    }
  }

  // 发射子弹
  private shoot(): void {
    if (!this.bulletPool) return;

    const bullet = this.bulletPool.get();
    bullet.transform.position.set(0, 0.5, 0);
    
    const script = bullet.getComponent(BulletScript);
    if (script) {
      script.direction = new Vector3(0, 0, 1); // 设置方向
    }
  }
}
```

## 4. 关键设计要点

### 4.1 避免循环依赖

**问题**：`createFn` 中引用 `this.bulletPool` 时，pool 还未赋值。

```typescript
// ❌ 错误：循环依赖
private initPools(): void {
  this.bulletPool = new ObjectPool<Entity>(
    () => {
      const bullet = this.createBullet();
      bullet.getComponent(BulletScript).setPool(this.bulletPool!); // 此时为 null!
      return bullet;
    },
    ...
  );
}
```

**解决**：在 `resetFn` 中设置 pool 引用。

```typescript
// ✅ 正确：延迟设置 pool
private initPools(): void {
  this.bulletPool = new ObjectPool<Entity>(
    () => this.createBullet(),  // 不设置 pool
    (bullet) => {
      const script = bullet.getComponent(BulletScript);
      if (script) {
        script.setPool(this.bulletPool!);  // 在 reset 时设置
        script.reset();
      }
    },
    30
  );
}
```

### 4.2 对象生命周期

```
创建 (createFn)
   ↓
初始化状态 (resetFn) ← 获取对象时调用
   ↓
使用 (enabled = true)
   ↓
回收 (return) → enabled = false, 移出视野
   ↓
复用 (回到对象池等待下次 get)
```

### 4.3 resetFn 的职责

- **设置 pool 引用**：`script.setPool(this.bulletPool!)`
- **重置对象状态**：重置生命值、位置、方向等
- **重置标记**：如 `isDead = false`

## 5. 多类型对象池管理

```typescript
class GameManager extends Script {
  private pools: Map<string, ObjectPool<Entity>> = new Map();

  createPool<T extends Entity>(
    name: string,
    createFn: () => T,
    resetFn: (obj: T) => void,
    initialSize: number = 10
  ): void {
    this.pools.set(name, new ObjectPool(createFn, resetFn, initialSize));
  }

  get(name: string): Entity | null {
    const pool = this.pools.get(name);
    return pool ? pool.get() : null;
  }

  return(name: string, obj: Entity): void {
    const pool = this.pools.get(name);
    if (pool) pool.return(obj);
  }
}
```

## 6. 完整游戏示例：射击游戏对象池

```typescript
class GameManager extends Script {
  private bulletPool!: ObjectPool<Entity>;
  private enemyPool!: ObjectPool<Entity>;

  onAwake(): void {
    this.initPools();
  }

  private initPools(): void {
    // 子弹池
    this.bulletPool = new ObjectPool<Entity>(
      () => this.createBulletBase(),
      (bullet) => {
        const script = bullet.getComponent(BulletScript);
        if (script) {
          script.setPool(this.bulletPool);
          script.reset();
        }
      },
      30
    );

    // 敌人池
    this.enemyPool = new ObjectPool<Entity>(
      () => this.createEnemyBase(),
      (enemy) => {
        const script = enemy.getComponent(EnemyScript);
        if (script) {
          script.setPool(this.enemyPool);
          script.reset();
        }
      },
      20
    );
  }

  // 发射子弹
  shoot(position: Vector3, direction: Vector3): void {
    const bullet = this.bulletPool.get();
    bullet.transform.position = position;
    
    const script = bullet.getComponent(BulletScript);
    if (script) {
      script.direction = direction;
    }
  }

  // 生成敌人
  spawnEnemy(position: Vector3): void {
    const enemy = this.enemyPool.get();
    enemy.transform.position = position;
  }
}
```

## 7. 常见问题

### Q: 对象回收后仍然可见？
A: 确保 `return()` 中设置了 `obj.enabled = false` 并移出视野。

### Q: Pool 引用为 null？
A: 检查是否在 `resetFn` 中正确调用了 `setPool()`，且 `get()` 时调用了 `resetFn()`。

### Q: 对象池性能优化？
A: 预创建足够数量的对象，避免运行时再创建。
