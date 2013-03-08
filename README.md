# Html Elements Framework & Thcydides Integration Patch

Этот патч позволяет интегрировать фреймворк 
[HtmlElements](https://github.com/yandex-qatools/htmlelements/) 
с фреймоврком 
[Thucydides](https://github.com/thucydides-webtests/thucydides/)

Так как это патч, то использовать его нужно очень аккуратно. 

## Agenda

По сути, патч затрагивает всего два файла.

* Во первых, он добавляет `htmlelements-all` в зависимости модуля `thucydides-core`. 
Для этого мы прописываем в 
[pom.xml](https://github.com/thucydides-webtests/thucydides/blob/master/thucydides-core/pom.xml)
следующие строки:

```diff
--- a/thucydides-core/pom.xml
+++ b/thucydides-core/pom.xml
@@ -50,6 +50,17 @@
     </build>
     <dependencies>
         <dependency>
+            <groupId>ru.yandex.qatools.htmlelements</groupId>
+            <artifactId>htmlelements-all</artifactId>
+            <version>1.9</version>
+            <exclusions>
+                <exclusion>
+                    <groupId>org.seleniumhq.selenium</groupId>
+                    <artifactId>selenium-java</artifactId>
+                </exclusion>
+            </exclusions>
+        </dependency>
+        <dependency>
             <groupId>cglib</groupId>
             <artifactId>cglib</artifactId>
             <version>2.2.2</version>
```

Так как в thucydides прописаны зависимости от конкретной версии `selenium-java`, 
а `htmlelements-all` совместим со всеми версиями selenium, то мы эксклюдим его из зависимостей (мы так наивно считаем:).

* Модифицируем класс 
[WebDriverFactory](https://github.com/thucydides-webtests/thucydides/blob/master/thucydides-core/src/main/java/net/thucydides/core/webdriver/WebDriverFactory.java)
следующим образом: 

```diff
     public static void initElementsWithAjaxSupport(final PageObject pageObject, final WebDriver driver) {
-        ElementLocatorFactory finder = getElementLocatorFactorySelector().getLocatorFor(driver);
-        PageFactory.initElements(finder, pageObject);
-        initWebElementFacades(new WebElementFacadeFieldDecorator(finder), pageObject, driver);
+        HtmlElementLoader.populatePageObject(pageObject, driver);
     }
 
     private static ElementLocatorFactorySelector getElementLocatorFactorySelector() {
@@ -572,10 +571,7 @@ public class WebDriverFactory {
     }
 
     public static void initElementsWithAjaxSupport(final PageObject pageObject, final WebDriver driver, int timeoutInSeconds) {
-        ElementLocatorFactory finder = getElementLocatorFactorySelector().withTimeout(timeoutInSeconds).getLocatorFor(driver);
-        PageFactory.initElements(finder, pageObject);
-        initWebElementFacades(new WebElementFacadeFieldDecorator(finder), pageObject, driver);
-
+        HtmlElementLoader.populatePageObject(pageObject, driver);
     }
```

Джон Смарт использует собственный `ElementLocatorFactory`, принцип выбора которого можно посмотреть в классе 
[ElementLocatorFactorySelector](https://github.com/thucydides-webtests/thucydides/blob/master/thucydides-core/src/main/java/net/thucydides/core/webdriver/ElementLocatorFactorySelector.java).
В зависимости от настроек фреймворка выбирается один из следующих реализаций: 

- Thycidides [DisplayedElementLocatorFactory](https://github.com/thucydides-webtests/thucydides/blob/master/thucydides-core/src/main/java/net/thucydides/core/webdriver/DisplayedElementLocatorFactory.java)
- Selenium [AjaxElementLocatorFactory](http://code.google.com/p/selenium/source/browse/java/client/src/org/openqa/selenium/support/pagefactory/AjaxElementLocatorFactory.java)
- Selenium [DefaultElementLocatorFactory](http://code.google.com/p/selenium/source/browse/java/client/src/org/openqa/selenium/support/pagefactory/DefaultElementLocatorFactory.java)

Таким образом, получается, что фреймворк можно сконфигурировать таким образом, чтобы использовать стандартные локаторы из PageObject.
А вот по дефолту, использутеся собственная реализация Джона Смарта. 

Если по простому, то `DisplayedElementLocatorFactory` делает так, что пользователя отсутсиве элемента в DOM-структуре
и его невидимость для пользователя = одно и то же. С одной стороны, это удобно, так как пользователь все равно этот 
элемент не видит, с друой стороны это противоречит закону зайца: 

- Ты видишь зайца?
- Нет
- А он есть.

В общем-то, это дело вкуса. Мы же используем по дефолту 
[AjaxElementLocator](https://github.com/yandex-qatools/htmlelements/blob/master/htmlelements-java/src/main/java/ru/yandex/qatools/htmlelements/pagefactory/AjaxElementLocator.java),
немного модифицирвоанный под концецию htmlelements. 

В общем, при использовании этого патча в ваших тестах могут появится такие спец.эффекты.

How To
======

Для того, чтобы использовать пропатченную версию Thucydides, нужно сделать следующие шаги:
  1. Выкачиваем Thucydides: 

     ```shell
     $ git clone git@github.com:thucydides-webtests/thucydides.git
     ```

  2. Выкачиваем патч:

     ```shell
     $ git clone git@github.com:yandex-qatools/htmlelements-thucydides-integration-patch.git patch
     ```

  3. Заходим в корневую директорию проекта Thucydides:

     ```shell
     $ cd thucydides
     ```

  3. Делаем checkout в ветку со свежей стабильной версией (на текущий момент 0.9.100):

     ```shell
     $ git checkout release-0.9.100
     ```
     
  3. Применяем патч htmlelements_integration.patch:

     ```shell
     $ git am < ../patch/htmlelements_integration.patch
     ```
     
  4. Меняем версии в pom-файлах, чтобы не перекрывать непропатченную версию, например на 0.9.100-PATCHED.
     
  5. Деплоим в мавен-репозитрий, чтобы можно было подключать к своим проектам.

Теперь при подключении к проекту Thucydides из мавен-репозитория будет подтягиваться пропатченная версия.

