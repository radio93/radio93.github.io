---
title: MVC模式
date: 2020-11-06 14:13:27
tags:
- java
- 设计模式
---

# MVC模式

>MVC 模式代表 Model-View-Controller（模型-视图-控制器） 模式。这种模式用于应用程序的分层开发。
>
>**Model（模型）** - 模型代表一个存取数据的对象或 JAVA POJO。它也可以带有逻辑，在数据变化时更新控制器。
>
>**View（视图）** - 视图代表模型包含的数据的可视化。
>
>**Controller（控制器）** - 控制器作用于模型和视图上。它控制数据流向模型对象，并在数据变化时更新视图。它使视图与模型分离开。

##### 示例演示：

1. 新建一个model

   ```java
   package test.mvc;
   
   import lombok.Data;
   
   @Data
   public class PeopleModel {
   
       private Integer id;
       private String name;
   }
   ```

2. 新建view

   ```java
   package test.mvc;
   
   public class PeopleView {
       void showPeople(PeopleModel model) {
           System.out.println(model.toString());
       }
   }
   ```

3. 新建controller

   ```java
   package test.mvc;
   
   public class PeopleController {
       private PeopleModel model;
       private PeopleView view;
   
       public PeopleController(PeopleModel model, PeopleView view) {
           this.model = model;
           this.view = view;
       }
   
       public void setPeopleId(Integer id){
           model.setId(id);
       }
   
       public Integer getPeopleId(){
           return model.getId();
       }
   
       public void setPeopleName(String name){
           model.setName(name);
       }
   
       public String getPeopleName(){
           return model.getName();
       }
       void showPeople(){
           view.showPeople(model);
       }
   }
   ```

4. 完成上述步骤，我们就可以开始演示了

   ```java
   package test.mvc;
   
   public class MvcDemo {
       static PeopleModel getModel() {
           PeopleModel model = new PeopleModel();
           model.setId(1);
           model.setName("abc");
           return model;
       }
   
       public static void main(String[] args) {
           //模拟数据库取值
           PeopleModel model = getModel();
           //创建视图
           PeopleView view = new PeopleView();
           //把people信息输出给控制台
           PeopleController controller = new PeopleController(model, view);
           controller.showPeople();
   
           //更新数据
           controller.setPeopleId(2);
           controller.setPeopleName("def");
           //再次输出
           controller.showPeople();
       }
   }
   ```

   运行结果：

   ```java
   PeopleModel(id=1, name=abc)
   PeopleModel(id=2, name=def)
   ```

##### 说明：

> MVC模式是典型的前后端数据交互模式。