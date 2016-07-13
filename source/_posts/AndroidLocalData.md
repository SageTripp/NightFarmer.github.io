---
title: android轻量级本地数据存储的实现
date: 2016-05-30 15:26:17
tags: [Android, 数据存储]
category: Android
---

android应用程序常用的本地数据存储一般用数据库和SharedPreferences，或者用文件系统缓存一些块状的数据如图片音视频等。
而SharedPreferences一般情况下只能存储一些基础数据类型的键值对，而本地存储存一些对象或者集合类的时候只能选择使用sqlite数据库。
对于上面这些情况确实可用满足我们的日常开发工作，但是对于存储一些少量的本地对象数据，如登陆人信息这样的，若采用SharedPreferences以键值对的形式存贮对象的各个属性则会显得非常繁琐，若使用数据库存储这个数据则会让代码显得非常臃肿。<!-- more -->

所以呢，需要有一种更加轻便的方式来存贮碎片化的数据。

数据库、SharedPreferences以及文件系统存储的本质其实都是将数据存储到文件中，只是提供的存取方式不同。
数据库以关系型数据库的设计和sql来进行数据库的存取操作；
SharedPreferences提供键值对的方式进场存取，以xml文件的形式来进行文本存取；
而文件系统以流的形式存取数据。

在我们的应用场景中需要一种数据库和SharedPreferences折中的复杂度的方式来存取数据，如果在这两种的基础上进行扩展，则有两种方式来实现，
一种是对sqlite进行orm封装，一种是对SharedPreferences的扩展
orm框架的话是存在非常多的开源的库的，这里我们就不重复早轮子了，而是选择使用SharedPreferences扩展一种对对象的存取方式。

先上代码
```java
public class LocalData {
    private String fileName;
    private SharedPreferences sp;
    public static final String DEFAULT = "default";

    public LocalData(Context context) {
        this(context, "default");
    }

    public LocalData(Context context, String fileName) {
        this.fileName = fileName;
        this.sp = context.getSharedPreferences(fileName, 0);
    }

    public void put(Serializable ser) {
        this.put("default", ser);
    }
   
    public void put(String key, Serializable ser) {
        try {
            if(ser == null) {
                this.sp.edit().remove(key).commit();
            } else {
                byte[] var5 = ByteUtil.objectToByte(ser);
                this.put(key + "-" + ser.getClass().getName(), HexUtil.encodeHexStr(var5));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public void put(String key, String value) {
        if(value == null) {
            this.sp.edit().remove(key).commit();
        } else {
            this.sp.edit().putString(key, value).commit();
        }
    }

    public <T> T get(Class<T> cls) {
        return this.get("default", cls);
    }

    public <T> T get(String key, Class<T> cls, Cipher cipher) {
        Object obj = null;
        try {
            String e = this.get(key + "-" + cls.getName(), (String)null);
            if(e == null) {
                return null;
            }
            byte[] bytes = HexUtil.decodeHex(e.toCharArray());
            obj = ByteUtil.byteToObject(bytes);
        } catch (Exception var7) {
            var7.printStackTrace();
        }

        return obj;
    }

    public String get(String key, String defValue) {
        return this.sp.getString(key, defValue);
    }
}
```
上面这段代码中核心提供了两个方法，一个put方法用于存储对象，一个get方法用于读取对象
在读取和存贮过程中分别将对象序列化为字节数组并将字节数组转换为字符串以及逆向过程
在对象转换为字符串之后则是通过SharedPreferences对数据进行存取操作，这一过程看上去并不复杂。
而存取的key则使用class的name来和指定的key拼接进行判断。

下面补上对象转字节数组以及字节数组转字符串以及逆向过程的代码：
```java
public class ByteUtil {
    public ByteUtil() {
    }

    public static Object byteToObject(byte[] bytes) throws Exception {
        ObjectInputStream ois = null;

        Object var2;
        try {
            ois = new ObjectInputStream(new ByteArrayInputStream(bytes));
            var2 = ois.readObject();
        } finally {
            if(ois != null) {
                ois.close();
            }

        }

        return var2;
    }

    public static byte[] objectToByte(Object obj) throws Exception {
        ObjectOutputStream oos = null;

        byte[] var3;
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            oos = new ObjectOutputStream(bos);
            oos.writeObject(obj);
            var3 = bos.toByteArray();
        } finally {
            if(oos != null) {
                oos.close();
            }

        }

        return var3;
    }
}

```

```java
public class HexUtil {
    private static final char[] DIGITS_LOWER = new char[]{'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f'};
    private static final char[] DIGITS_UPPER = new char[]{'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'};

    public HexUtil() {
    }

    public static char[] encodeHex(byte[] data) {
        return encodeHex(data, true);
    }

    public static char[] encodeHex(byte[] data, boolean toLowerCase) {
        return encodeHex(data, toLowerCase?DIGITS_LOWER:DIGITS_UPPER);
    }

    protected static char[] encodeHex(byte[] data, char[] toDigits) {
        int l = data.length;
        char[] out = new char[l << 1];
        int i = 0;

        for(int j = 0; i < l; ++i) {
            out[j++] = toDigits[(240 & data[i]) >>> 4];
            out[j++] = toDigits[15 & data[i]];
        }

        return out;
    }

    public static String encodeHexStr(byte[] data) {
        return encodeHexStr(data, true);
    }

    public static String encodeHexStr(byte[] data, boolean toLowerCase) {
        return encodeHexStr(data, toLowerCase?DIGITS_LOWER:DIGITS_UPPER);
    }

    protected static String encodeHexStr(byte[] data, char[] toDigits) {
        return new String(encodeHex(data, toDigits));
    }

    public static byte[] decodeHex(char[] data) {
        int len = data.length;
        if((len & 1) != 0) {
            throw new RuntimeException("Odd number of characters.");
        } else {
            byte[] out = new byte[len >> 1];
            int i = 0;

            for(int j = 0; j < len; ++i) {
                int f = toDigit(data[j], j) << 4;
                ++j;
                f |= toDigit(data[j], j);
                ++j;
                out[i] = (byte)(f & 255);
            }

            return out;
        }
    }

    protected static int toDigit(char ch, int index) {
        int digit = Character.digit(ch, 16);
        if(digit == -1) {
            throw new RuntimeException("Illegal hexadecimal character " + ch + " at index " + index);
        } else {
            return digit;
        }
    }
}
```

> 注意：被存取的这些的对象在实现Serializable接口的同时必须要指定serialVersionUID，否则修改类结构之后将会视为两个不同的类而造成反序列化失败，如：
```java
private static final long serialVersionUID = 1;
```



