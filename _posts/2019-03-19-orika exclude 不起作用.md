---
layout: post
title: "orika exclude 不起作用"
date: 2019-03-19 20:00:00
tags: orika java
---

在使用 orika 进行 map 转 bean 发现 exclude 方法不能有效的排除不需要转换的属性.最后经查看源码发现是 orika 的一个bug.

下面记录一下是如何发现这个问题.




## 直接看使用 orika 的代码

很显然这里想排除 map 中的 b 属性. 转换后B对象b属性的值应该为null.但结果并不是.

```java
public class Issue314TestCase {

	@Test
	public void exclude_classMapBuilderForMaps() {

		DefaultMapperFactory mapperFactory = new DefaultMapperFactory.Builder().build();
		mapperFactory.classMap(Map.class, B.class).exclude("b").byDefault().register();

		Map<String, Object> map = new HashMap<String, Object>();
		map.put("a", "a");
		map.put("b", "b");

		B b = mapperFactory.getMapperFacade().map(map, B.class);
		assertNull(b.getB());
	}

	public static class B {
		private String a;
		private String b;

		public String getA() {
			return a;
		}

		public void setA(String a) {
			this.a = a;
		}

		public String getB() {
			return b;
		}

		public void setB(String b) {
			this.b = b;
		}
	}

}
```
## 查看 exclude 方法源码

只找到了一个方法,在 ClassMapBuilder 类中

```java
    /**
     * Exclude the specified field from mapping
     * 
     * @param fieldName
     *            the name of the field/property to exclude
     * @return this ClassMapBuilder
     */
    public ClassMapBuilder<A, B> exclude(String fieldName) {
        if (propertyResolver.existsProperty(getAType(), fieldName) && propertyResolver.existsProperty(getBType(), fieldName)) {
            return fieldMap(fieldName).exclude().add();
        } else {
            return this;
        }
    }
```
看到这段代码,猜测一下这个逻辑应该是,排除的属性(fieldName)必须在 A 类和 B 中都存在. 然后断点在 if 条件上看看 A 类和 B 类中的属性都有什么?

- map 的属性在 propertyCache 中是这样的
![Amapclass](https://github.com/yupengj/yupengj.github.io/blob/master/images/Amapclass.png?raw=true) 

- Bean 类的属性在 propertyCache 中是这样的
![Bclass](https://github.com/yupengj/yupengj.github.io/blob/master/images/Bclass.png?raw=true) 

所以第一个条件 `propertyResolver.existsProperty(getAType(), fieldName)`是永远不会成立的. 这是什么鬼 map 中的属性排除不支持吗?

## 发现 ClassMapBuilderForMaps 类.

找了好久,各种怀疑,最后看了一下 Variables 中的 this !!! 
![ClassMapThis](https://github.com/yupengj/yupengj.github.io/blob/master/images/ClassMapThis.png?raw=true)

原来 `mapperFactory.classMap(Map.class, B.class)` 创建出来的是 ClassMapBuilderForMaps 类,继承 ClassMapBuilder 类.
那为什么创建出来 ClassMapBuilderForMaps 类呢? 猜一下肯定是 classMap 方法中有一个参数是 Map. 源码如下:

```java
/**
 * Factory produces instances of ClassMapBuilderForMaps
 */
public static class Factory extends ClassMapBuilderFactory {

    @Override
    protected <A, B> boolean appliesTo(Type<A> aType, Type<B> bType) {
        return (aType.isMap() && !bType.isMap()) || (bType.isMap() && !aType.isMap());
    }

	/* (non-Javadoc)
	 * @see ma.glasnost.orika.metadata.ClassMapBuilderFactory#newClassMapBuilder(ma.glasnost.orika.metadata.Type, ma.glasnost.orika.metadata.Type, ma.glasnost.orika.property.PropertyResolverStrategy, ma.glasnost.orika.DefaultFieldMapper[])
	 */
    @Override
	protected <A, B> ClassMapBuilder<A,B> newClassMapBuilder(
			Type<A> aType, Type<B> bType,
			MapperFactory mapperFactory,
			PropertyResolverStrategy propertyResolver,
			DefaultFieldMapper[] defaults) {
		
		return new ClassMapBuilderForMaps<A,B>(aType, bType, mapperFactory, propertyResolver, defaults);
	}
}
```
这个类是 ClassMapBuilderForMaps 类中的静态内部类, 其中 appliesTo 方法就是创建 ClassMapBuilderForMaps 类的条件.
也就是, A 和 B 有且只有一个类是 Map. 另一个类是 Bean

找到这里问题也就发现了. 
1. ClassMapBuilderForMaps 类对于 exclude 方法的逻辑应该是: 要排除的属性在 A 或 B 某一个类中存在就可以,不能像父类方法一样要求要排除的属性在 A 和 B 类中都存在. 
2. 所以要重写父类 ClassMapBuilder 中的 exclude 方法. 重写后的代码 :

```java
/**
     * Exclude the specified field from bean mapping
     * 
     * @param fieldName
     *            the name of the field/property to exclude
     * @return this ClassMapBuilder
     */
    @Override
    public ClassMapBuilder<A, B> exclude(String fieldName) {
    	Type<?> type = isATypeBean() ? getAType() : getBType();
    	if(getPropertyResolver().existsProperty(type, fieldName)) {
    		 return fieldMap(fieldName).exclude().add();
    	}else {
    		return this;
    	}
    }
```
上面这段代码就是 ClassMapBuilderForMaps 类中缺少的代码. 然后我把这段代码 PR 到 orika 的 github 仓库,不到2分钟他们就 Merged 到主线上了.

但是现在我们使用的是以前的版本,要怎么绕过这个问题? 可以去掉 byDefault 方法用 ClassMapBuilder 的 field 方法指定要转换的属性,这样做在属性较多时会麻烦一些

## 相关类
[ClassMapBuilder.java](https://github.com/orika-mapper/orika/blob/master/core/src/main/java/ma/glasnost/orika/metadata/ClassMapBuilder.java)
[ClassMapBuilderForMaps](https://github.com/orika-mapper/orika/blob/master/core/src/main/java/ma/glasnost/orika/metadata/ClassMapBuilderForMaps.java)