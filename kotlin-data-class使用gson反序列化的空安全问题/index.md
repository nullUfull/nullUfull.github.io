# kotlin data class使用Gson反序列化的空安全问题

## 缘起
最近遇到一个NPE，是使用Kotlin定义的一个非空的浮点集合`List<Float>`，但是在具体业务侧取用的时候，出现了如下Crash：
```
java.lang.NullPointerException: Attempt to invoke virtual method 'float java.lang.Number.floatValue()' on a null object reference
```
从非空集合中取到了null，导致出现了NPE，经过一番溯源，发现这个集合是从数据库中获取的Json串通过Gsono反序列化出来的，而Gson反序列化的时候并不会报错，直到数据使用时才抛出NPE。

```
val type = object : TypeToken<List<Float>>() {}.type
Gson().fromJson<List<Float>>("[1.0,2.0,null]",type)
```

## 觅踪
内部都是通过TypeAdapter来处理序列化和反序列化的逻辑的，例如我的`List<Float>`则存在一个`CollectionTypeAdapterFactory`来获取一个集合的适配器来处理，
而具体的元素类型又交付给`ObjectTypeAdapter`来处理，其中如果读到null则直接返回null了，并没有做其余的处理。

```
public final class CollectionTypeAdapterFactory implements TypeAdapterFactory {
  private final ConstructorConstructor constructorConstructor;

  public CollectionTypeAdapterFactory(ConstructorConstructor constructorConstructor) {
    this.constructorConstructor = constructorConstructor;
  }

  @Override
  public <T> TypeAdapter<T> create(Gson gson, TypeToken<T> typeToken) {
   ...
  private static final class Adapter<E> extends TypeAdapter<Collection<E>> {
    ...

    @Override public Collection<E> read(JsonReader in) throws IOException {
      if (in.peek() == JsonToken.NULL) {
        in.nextNull();
        return null;
      }

      Collection<E> collection = constructor.construct();
      in.beginArray();
      while (in.hasNext()) {
        E instance = elementTypeAdapter.read(in); // 1. 读取集合中的元素
        collection.add(instance);
      }
      in.endArray();
      return collection;
    }

    ...
}

public final class ObjectTypeAdapter extends TypeAdapter<Object> {
    ...
  @Override public Object read(JsonReader in) throws IOException {
    JsonToken token = in.peek();
    switch (token) {
    ...
    case NUMBER:
      return in.nextDouble();

    case BOOLEAN:
      return in.nextBoolean();

    case NULL: // 2. 如果是null则返回null
      in.nextNull();
      return null;

    default:
      throw new IllegalStateException();
    }
  }
  ...

```

阅读源码后我们也就可以知道Gson对Kotlin的一些特性别并不兼容：w
1. 不支持空安全
2. 不支持默认参数

## 兼容
Gson是并没有适配Kotlin的可空类型的，会返回一个带有null的集合，导致取用的时候会NPE，但是查看了相关写入数据库的代码也并咩有找到存在写入null的情况，因此我们只能在反序列化的时候做一个兼容逻辑，如果读取到null值则则写入默认值0f到集合，以及为了进一步排查问题，也可以打印相关的日志，比如一个集合中存在多少个null值，以便可以进一步分析。

不过也有一个Square出品的与kotlin兼容极好的现代化JSON解析库——[moshi](https://github.com/square/moshi)

针对于List<Float>类型，我们注入一个自定义的类型适配器，Gson内部会取类型所对应的Adapter来处理序列化和反序列化。

### 编写List<Float>的自定义适配器
```
class FloatListTypeAdapter : TypeAdapter<List<Float>>() {
    override fun write(writer: JsonWriter?, value: List<Float>?) {
        writer ?: return
        val list = value ?: emptyList()
        writer.beginArray()
        list.forEach {
            writer.value(it)
        }
        writer.endArray()
    }

    override fun read(reader: JsonReader?): List<Float> {
        reader ?: return emptyList()
        reader.beginArray()
        val list = mutableListOf<Float>()
        var nullValueCount = 0
        while (reader.hasNext()) {
            if (reader.peek() == JsonToken.NULL) {
                nullValueCount++
                list.add(0f)
                reader.nextNull()
                continue
            }
            list.add(reader.nextDouble().toFloat())
        }
        reader.endArray()
        if (nullValueCount != 0) {
            Logger.e(TAG, "[read] nullCount = $nullValueCount")
        }
        return list
    }

    companion object {
        private const val TAG = "FloatListTypeAdapter"
    }
}
```
### 注册FloatListAdapter
```
GsonBuilder().registerTypeAdapter(
        object : TypeToken<List<Float>>() {}.type,
        FloatListTypeAdapter()
    ).create()
```
