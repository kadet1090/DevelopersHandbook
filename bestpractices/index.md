# FreeCAD - Code Review Book

This document aims to provide set of good practices that should be helpful for both developers and code reviewers. They should be treated like food recipies - you can play with them, alter them - but every change should be thoughtful and intentional.

This document is __very__ much inspired by the [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).
Most rules presented here are from this document and whenever something is not covered or you are in doubt - 
don't hesitate to consult it. 

> [!NOTE]  
> Remember that code review is a collaborative discussion. Don’t hesitate to ask for clarification or help when needed. Reviewers can also make mistakes, the goal is to work together to refine the code to a point where everyone is satisfied.

While this guideline might not be consistently followed throughout all of the existing codebase, adhering to these practices moving forward will help improve the overall quality of the code and make future contributions more maintainable.

In this document the **bolded** text will indicate how important each suggestion is. 
 - **must** will be used for fundamental things that should be non-controversial and which you really should follow
 - **should** will be used for important details that will apply for vast majority of cases, there could however be valid reasons to ignore them depending on context
 - **could** will be used for best practices, things that you should try to follow but not following them is not an error per se

## Common Rules
1. Consistency **must** be preferred over strict rule following. If, for example, in a given context code uses different naming scheme - follow it instead of one described in that document.
2. The aim of Code Review **is not** to find errors in code but to ensure code quality.
3. Reviewers **should** comment mostly on added code and discuss existing one only if making change to existing code would help the new fragment.
4. Comments **could** be made to existing comments just to note that other (potentially better) soluions are available and should be used instead when writing new code.
5. Reviewer **can** hint to make more changes to existing code in order to improve the propose solution (for example, refactor another class so it can be used)
6. Code **must** follow the Scout Rule - https://biratkirat.medium.com/step-8-the-boy-scout-rule-robert-c-martin-uncle-bob-9ac839778385 - i.e. leave code in better shape than you found it.
7. PRs **must not** contain any remaints of development code, like debug statements other than actual logs.
8. New code **must not** introduce any new linter warnings
9. You **should** have `pre-commit` installed and working,.

## Basic Code Rules
1. (C++ only) New code **must** be formatted with clang-format tool or in a way that is compatible with clang-format result if file is excluded from auto formatting.
1. (Python only) Code **must** follow the PEP 8 standard and formatted with Black.
3. Main execution path **should** be the least indented one, i.e. conditions should cover specific cases.
4. Early-Exit **should** be preferred to prune unwanted execution branches fast.
5. ?? Classes that do provide business logic **should** be stateless. This helps with reusability.
6. ?? Functions / Methods **should** be pure i.e. they result should only depend on arguments (and object state in case of method). This helps with reausability, predictability and testing.
7. Global state (global variables, static fields, singletons) **should be** avoided.
8. Code **should** be written in a way that it expresses intent, i.e. what should be done, rather than just how it is done. 
    <details>
        <summary>Example #1</summary>
        Consider this code:
        ```c++
            void setOverlayMode(OverlayMode mode)
            {
                // ... some code ...

                QDockWidget *dock = nullptr;
                
                for (auto w = qApp->widgetAt(QCursor::pos()); w; w = w->parentWidget()) {
                    dock = qobject_cast<QDockWidget*>(w);
                    if (dock) {
                        break;
                    }
                    auto tabWidget = qobject_cast<OverlayTabWidget*>(w);
                    if (tabWidget) {
                        dock = tabWidget->currentDockWidget();
                        if (dock) {
                            break;
                        }
                    }
                }

                if (!dock) {
                    for (auto w = qApp->focusWidget(); w; w = w->parentWidget()) {
                        dock = qobject_cast<QDockWidget*>(w);
                        if (dock) {
                            break;
                        }
                    }
                }

                // some more code ...

                toggleOverlay(dock, m);
            }
        ```
        
        It is hard to understand what is the job of the for loop inside `if (!dock)` statement. 
        We can refactor it to a new `QWidget* findClosestDockWidget()` method for it to look like this:
        
        ```c++
            void setOverlayMode(OverlayMode mode)
            {
                // ... some code ...

                QDockWidget *dock = findClosestDockWidget();

                // ... some more code ...

                toggleOverlay(dock, m);
            }
        ```
        The findClosestDockWidget() could either be implemented as private method or an inner function using lambdas.

        ```c++
        auto findClosestDockWidget = []() { ... }
        ```

        That way reading through code of `setOverlayMode` we don't need to care about the details of finding the closest dock widget.
    </details>
9. Immutable data **should** be used whenever possible, (For C++ use `const`)
    <details>
        <summary>Rationale</summary>
        It is much easier to reason about code that deals with data that does not change. 
    </details>
10. Boolean arguments **must** be avoided. Use enumerations instead - enum with 2 values is absolutely fine. 
    For python boolean arguments are ok, but they **must** forced to be keyword ones.
    <details>
        <summary>Example #1 (C++)</summary>
        Consider following example:
        
        ```c++
        mapper.populate(false, it.Key(), it.Value());
        ```

        It is impossible to understand what false means without consulting the documentation or at least the method signature.
        Instead the enum should be used:
        
        ```c++
        mapper.populate(MappingStatus::Modified, it.Key(), it.Value());
        ```

        Now the intent is clear
    </details>
11. Integers **must not** be used to express anything other than numbers. For enumerations enums **must** be used.
12. (C++ only) `enum class` **should** be preferred over normal enum whenever possible.
13. (C++ only) C++ libraries/structures **should** be preferred over C ones. i.e. use std::array/vector instead of C array. 
14. (C++ only) C++ standard library **should** be utilized if possible, especially the `<algorithms>` one.
    <details>
        <summary>Example #1</summary>
        
        Consider following code:
        ```c++
            std::set<int> vertexSet;
            for (auto &s : face.getSubShapes(TopAbs_VERTEX)) {
                int idx = shape.findShape(s) - 1;
                if (idx >= 0 && vertexSet.insert(idx).second) {
                    vertices.push_back(idx);
                }
            }
        ```

        You can rewrite it in a following way:
        ```c++
            for (auto &vertex : face.getSubShapes(TopAbs_VERTEX)) {
                int vertexId = shape.findShape(vertex);

                if (vertexId > 0) {
                    vertexSet.insert(vertexId);
                }
            }


            std::copy(vertexSet.begin(), vertexSet.end(), std::back_inserter(vertices));
        ```

        This way you split the responsibility of computing unique set with result preparation.
    </details>
15. (C++ only) Return values **should** be preferred over out arguments.
    a. For methods that can fail you can use `std::optional`
    b. For methods that return multiple values it may be better to either provide dedicated struct for result or use `std::tuple` with expression binding.
16. If expression is not obvious - it **should** be given a name by using variable.
    <details>
        <summary>Example #1</summary>
        TODO: Find some good example
    </details>


## Naming Things
1. Code symbols (classes, structs, methods, functions, variables...) **must** have names that are meaningful and gramatically correct.
2. Variables **should not** be named using abbreviations and/or 1 letter names. Iterator variables or math related ones like `i` or `u` are obviously not covered by this rule.
3. Names **must not** use the hungarian notation.
4. (C++ only) Classes/Structs **must** be written in `PamelCase`, underscores are allowed but should be avoided.
5. (C++ only) Class members **should** be written in `camelCase`, underscores are allowed but should be avoided.
6. (C++ only) Global functions **should** be written in `camelCase`, underscores are allowed but should be avoided.
7. (C++ only) Enum cases **should** use `PascalCase`

## Commenting the code
1. Good naming things **must** be preferred over commenting the code.
2. Comments that describe what code does **should** be avoided, instead comments **should** explain intended result.
3. All edge-cases in code **must** be described with comment describing when such edge-case can occur and why it is solved in certain way.
4. All "Hacks" **must** be described with how the hack works, why it is applied and when it no longer will be needed.
5. Commented code **must** be contain additional information on why it was commented out and when it is safe to remove it.
