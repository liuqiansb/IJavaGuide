##### transient关键字

> 将不需要序列化的属性前添加关键字transient，序列化对象的时候，这个属性就不会被序列化。

```java
public class TransientTest {
    @Test
    public void test() throws IOException, ClassNotFoundException {
        serializeDog();
        deserializeDog();
    }
    public void serializeDog() throws IOException {
        Dog dog = new Dog("1","2");
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("G:\\Study\\src\\test\\java\\com\\kiy\\test\\1"));
        objectOutputStream.writeObject(dog);
    }
    public void deserializeDog() throws IOException, ClassNotFoundException {
        File file = new File("G:\\Study\\src\\test\\java\\com\\kiy\\test\\1");
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(file));
        Dog dog = (Dog) objectInputStream.readObject();
        System.out.println(dog);
    }
}
@AllArgsConstructor
@Data
class Dog implements Serializable {
    private static final long serialVersionUID = 123456L;
    private  String name;
    // 序列化时将忽略该属性
    private transient String age;
}
```

##### transient的底层实现原理

> 通过