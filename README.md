# java17migration
Tests and solutions for migration problems from java 8 to java 17.

This is not a general discussion of migration problems but
only deals with problems in our software.

## Migration Changes Applied

### Build Configuration (`build.xml`)

- Added `release="17"` to `<javac>` task to target Java 17 bytecode
- Added `includeantruntime="false"` to avoid Ant runtime classpath warnings
- Fixed compile classpath to include all dependency JARs from `lib/`
- Added `java9settings` target with conditional `--add-opens` JVM flags for Java 9+
- Added `--add-opens java.base/java.util=ALL-UNNAMED` JVM args to both `junit` and `java` tasks
- Added `fork="true"` to `hello` target's `<java>` task (required for JVM args)

### Dependency Updates (`project.properties`)

| Dependency     | Old Version | New Version | Reason                              |
|----------------|-------------|-------------|-------------------------------------|
| commons-lang3  | 3.12.0      | 3.14.0      | Latest stable, Java 17 compatible   |
| ivy            | 2.4.0       | 2.5.2       | Latest stable                       |
| groovy         | 4.0.5       | 4.0.15      | Bug fixes for Java 17 runtime       |
| slf4j          | 2.0.3       | 2.0.9       | Latest stable 2.x                   |
| velocity       | 2.3         | 2.3         | Already Java 17 compatible          |
| junit          | 4.13.2      | 4.13.2      | Already latest JUnit 4              |
| hamcrest       | 2.2         | 2.2         | Already latest                      |

### Java Source Code Changes

- Fixed raw generic types (`Class` → `Class<?>`, `Map` → `Map<String, Object>`)
- Updated Velocity engine configuration from deprecated 1.x property names to 2.x format:
  - `resource.loader` → `resource.loaders`
  - `string.resource.loader.class` → `resource.loader.string.class`
  - `string.resource.loader.repository.static` → `resource.loader.string.repository.static`

### IntelliJ IDEA Config (`java17migration.iml`)

- Updated JUnit reference from 4.12 → 4.13.2
- Updated hamcrest reference from hamcrest-core 1.3 → hamcrest 2.2

## Problem areas

### Deprecated API

For instance "new Integer(...)" and similar is migrated to Integer.valueOf(...).

### Java 9's module system

This causes reflection based code to fail when accessing classes in the 
jdk, for instance:

```
Unable to make field private final java.util.Comparator java.util.TreeMap.comparator accessible: module java.base does not "opens java.util" to unnamed module @2d58d9ed

java.lang.reflect.InaccessibleObjectException: Unable to make field private final java.util.Comparator java.util.TreeMap.comparator accessible: module java.base does not "opens java.util" to unnamed module @2d58d9ed
at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:354)
at java.base/java.lang.reflect.AccessibleObject.checkCanSetAccessible(AccessibleObject.java:297)
at java.base/java.lang.reflect.Field.checkCanSetAccessible(Field.java:178)
at java.base/java.lang.reflect.Field.setAccessible(Field.java:172)
at com.infodesire.v20.pojo.Pojos.getFields(Pojos.java:338)
```

**Solution:** Add `--add-opens java.base/java.util=ALL-UNNAMED` JVM argument. This is implemented
in the `java9settings` target in `build.xml`, which conditionally applies the flag when Java 9+ is detected:

```xml
    <target name="java9settings">
    
        <condition property="addOpensPropertsPart1" value="--add-opens">
          <javaversion atleast="9"/>
        </condition>
        <property name="addOpensPropertsPart1" value="-Dummy1=1" />
        <condition property="addOpensPropertsPart2" value="java.base/java.util=ALL-UNNAMED">
          <javaversion atleast="9"/>
        </condition>
        <property name="addOpensPropertsPart2" value="-Dummy2=2" />
    
    </target>
```

### Error parsing java version numbers

commons-lang

```
Caused by: java.lang.NumberFormatException: multiple points
	at java.base/jdk.internal.math.FloatingDecimal.readJavaFormatString(FloatingDecimal.java:1914)
	at java.base/jdk.internal.math.FloatingDecimal.parseFloat(FloatingDecimal.java:122)
	at java.base/java.lang.Float.parseFloat(Float.java:476)
	at org.apache.commons.lang.SystemUtils.getJavaVersionAsFloat(SystemUtils.java:756)
	at org.apache.commons.lang.SystemUtils.<clinit>(SystemUtils.java:469)
```

**Solution:** Replace `org.apache.commons.lang.*` with `org.apache.commons.lang3.*` (version 3.14.0).

### New version of velocity

#### Deprecated configuration:

```
# old way (velocity 1.7)
resource.loader=memory
memory.resource.loader.class=com.infodesire.v20.templating.velocity.MemoryResourceLoader

# the new way (velocity 2+)
resource.loaders=memory
resource.loader.memory.class=com.infodesire.v20.templating.velocity.MemoryResourceLoader
```

**Solution:** All Velocity property names updated to 2.x format in `Main.java`.

#### Problem with hyphens in variable names:

```
#set( $ext-debug="1" )
```

will cause:

```
66   [main] ERROR velocity.parser  - 1664649690654-0: Encountered "-" at line 1, column 11.
Was expecting one of:
    "[" ...
    <WHITESPACE> ...
    <NEWLINE> ...
    "=" ...
```

**Solution:** Set variable in velocity properties:

```
parser.allow_hyphen_in_identifiers=true
```

#### Velocity 2 uses JVMs encoding instead of ISO-8859-15

Some characters will display incorrectly in rendered text.

**Solution:** Override `getResourceReader` in MemoryResourceLoader:

```java
  public Reader getResourceReader( String name, String encoding ) throws ResourceNotFoundException {
    try {
      return new InputStreamReader( getResourceStream( name ), "ISO-8859-15" );
    }
    catch( UnsupportedEncodingException ex ) {
      throw new RuntimeException( ex );
    }
  }
```
 