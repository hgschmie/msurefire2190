# Reproduce Maven surefire dependency problems (MSUREFIRE-2190)

maven surefire plugin struggles with optional dependencies for the main artifact and test code.

This repo contains three artifacts with identical code:

- thing1 builds a main artifact without JPMS
- thing2 builds a main artifact with a strong (`requires`) dependency on jakarta.annotation.
- thing3 builds a main artifact with a compile-only (`requires static`) dependency on jakarta annotations.

The code and its test classes are otherwise identical.

Running `mvn -DskipTests clean install` builds all three modules and the test code.

Running `mvn surefire:test` passes the tests in the first two modules but fails in the third.

The surefire plugin, when it executes tests using JPMS adds all
referenced modules to the module path (in this case the module under
test itself and the jakarta.annotations-api jar). It then adds the
main module using `--add-modules` and patches this module with the
test classes (so that the test classes execute as part of the module).

In case of a `requires` relationship, the JVM will find the required
JPMS module and add it as accessible. This is why the "thing2" tests
pass.

In case of a `requires static` relationship, the JVM will not add the
module transitively as it is considered a compilation-only
dependency. For the code under test to be able to access the classes
from `jakarta.annotation`, they must be declared as a module. However,
the test code only adds the module under test with `--add-modules`.


----

The fix for this is pretty simple but must be done in the surefire
plugin. Instead of adding `--add-modules <... module id under
test..>`, it needs to add `--add-modules ALL-MODULE-PATH`. Which is
correct, because the code is not looking for compile time only
dependencies but actual runtime dependencies where the code under
execution may also need to access optional runtime dependencies (see
https://nipafx.dev/java-modules-optional-dependencies/ for a slightly
longer explanation).
