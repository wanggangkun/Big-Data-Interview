现在基本上都是采用 maven 来进行开发管理，我有一个需求是需要把通过 maven 管理的 java 工程打成可执行的 jar 包，这样
也就是说必需把工程依赖的 jar 包也一起打包。而使用 maven 默认的 package 命令构建的 jar 包中只包括了工程自身的 
class 文件，并没有包括依赖的 jar 包。我们可以通过配置插件来对工程进行打包。

###二、maven-shade-plugin基本功能

maven-shade-plugin提供了两大基本功能：
1. 将依赖的jar包打包到当前jar包（常规打包是不会将所依赖jar包打进来的）；
2. 对依赖的jar包进行重命名（用于类的隔离）；

###三、shade打包过程
shade插件绑定在maven的package阶段，他会将项目依赖的jar包解压并融合到项目自身编译文件中。

举个例子：例如我们的项目结构是

com.gavinzh.learn.shade
    Main
	
假设我们依赖了一个jar包，他的项目结构是:

com.fake.test
    A
    B
那么shade会将这两个结构融合为一个结构:

com
    gavinzh.learn.shade
        Main
    fake.test
        A
        B
并将上述文件打成一个jar包。


Relocating Classes
Java 工程经常会遇到第三方 Jar 包冲突，使用 maven shade plugin 解决 jar 或类的多版本冲突。 maven-shade-plugin 在打包时，
可以将项目中依赖的 jar 包中的一些类文件打包到项目构建生成的 jar 包中，在打包的时候把类重命名。下面的配置将 org.codehaus.plexus.util jar 
包重命名为 org.shaded.plexus.util。


  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>2.4.3</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <relocations>
                <relocation>
                  <pattern>org.codehaus.plexus.util</pattern>
                  <shadedPattern>org.shaded.plexus.util</shadedPattern>
                  <excludes>
                    <exclude>org.codehaus.plexus.util.xml.Xpp3Dom</exclude>
                    <exclude>org.codehaus.plexus.util.xml.pull.*</exclude>
                  </excludes>
                </relocation>
              </relocations>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
