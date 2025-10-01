---
layout: post
date: 2022-12-02
title: "How to Create a Java Atom"
author: yegor256
---

There are "atoms" in [EO](https://www.eolang.org) language, which are objects implemented by
the runtime platform, not by a composition of other EO objects. Most
notable examples of atoms are `int.plus`, `float.times`, and
`bool.while`. Here is a quick intruction to creating your own
atoms.

<!--more-->

Let's say, this is an EO program that uses your atom `md5` (it
builds an [MD5](https://en.wikipedia.org/wiki/MD5) hash of a `string`):

```
+package org.example
+alias org.example.md5

[] > app
  QQ.io.stdout > @
    md5
      "Hello, world!"
```

Put it into `src/main/eo/org/example` directory and try to compile
using our [Maven Plugin](https://github.com/objectionary/eo/tree/master/eo-maven-plugin):

```
$ mvn clean compile
```

The compilation will fail, because there is no atom definition yet.
Create this file, with the atom and put it to `src/main/eo/org/example/md5.eo`:

```
+package org.example

[txt] > md5 /string
```

Then, create a Java implementation of this atom in
`src/main/java/EOorg/EOexample/EOmd5.java` file:

```
package EOorg.EOexample;
import org.eolang.*;
import java.security.*;

@XmirObject(oname = "md5")
public class EOmd5 extends PhDefault {
  public EOmd5(final Phi sigma) {
    super(sigma);
    this.add("txt", new AtFree());
    this.add(
      "φ",
      new AtComposite(
        this,
        rho -> {
          String txt = new Param(rho, "txt").strong(String.class);
          byte[] bytes = txt.getBytes("UTF-8");
          byte[] hash = MessageDigest.getInstance("MD5").digest(bytes);
          return new Data.ToPhi(new String(hash));
        }
      )
    );
  }
}
```

Here, the `EOorg.EOexample` is the Java package that contains
all atoms and objects of `org.example` EO package. The `EO` prefix
is used in order to enable EO naming inside Java name space.

The class `PhDefault` is the parent of all Java atoms and I strongly
recommend you use it too. You class should implement a single argument
constructor with a parameter of type `Phi`. If you don't have it,
there will be a runtime error by reflection API: EO runtime won't
be able to instantiate your class. The argument `sigma` you should pass
to the constructor of the parent class `PhDefault`.

Then, using `this.add()` you configure the attributes of the atom,
which you can later use inside the code encapsulated by the instance of the `AtComposite`
class.

The attribute you add with `this.add("φ")` is the "body" of the atom.
It will be evaluated when the atom will be dataized.

Then, using the class `Param` you can get the value of any incoming attribute
of your atom. The method `strong()` finds the attribute and dataizes it.

The `new Data.ToPhi()` is the best way to return the result back to EO.

The presence of `@XmirObject` annotation will help EO runtime to properly
name your atom in the logs (you can omit this annotation).

Now, let's create a unit test for the atom. Put this file
into `src/test/java/EOorg/EOexample/EOmd5Test.java`:

```
package EOorg.EOeolang;
import org.eolang.*;
import static org.hamcrest.*;
import org.junit.jupiter.api.Test;

public final class EOmd5Test {
  @Test
  public void calculatesHashString() {
    assertThat(
      new Dataized(
        new PhWith(
          new EOmd5(Phi.Φ),
          "txt",
          new Data.ToPhi("Hello, world!")
        )
      ).take(String.class),
      Matchers.equalTo("A6F5EC87EB4E9027295")
    );
  }
}
```

Then, make an EO test for your atom:

```
+alias org.eolang.hamcrest.assert-that
+junit
+package org.example

[] > calculates-hash-code
  assert-that > @
    md5
      "Hello, world!"
    $.equal-to
      "A6F5EC87EB4E9027295"
```

Run them both using our Maven Plugin:

```
mvn clean test
```

That's it.

<hr/>

You can find a good example of some atoms in the
[`eo-files`](https://github.com/objectionary/eo-files) repository.
