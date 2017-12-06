---
layout: post
title:  "Spring Boot, Mongo and Docker. Run all with a single command."
date:   2017-10-07 15:00:00
categories: [tutorial]
comments: true
---
In this article, we’ll se how to run a spring boot application with mongo db and a mongo client with a single command. For this purpose, we must dockerize our application, the mongo database and the mongo client.

### Spring Boot Application

To create our Spring Boot application we use the project creation wizard provided by the Spring web.

![Spring Initializr]({{ "/img/springinitializr.png" | absolute_url }})

Once the project is downloaded, we need to add the necessary configuration to dockerize the application and for the application to connect to Mongo.

### How to dockerize the application

We only need to add a plugin to the pom.xml file and our application will aready been dockerized.

{% highlight xml %}
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>dockerfile-maven-plugin</artifactId>
  <version>1.3.4</version>
  <configuration>
    <repository>${docker.image.prefix}/${project.artifactId}&nbsp; </repository>
  </configuration>
  <executions>
    <execution>
      <id>default</id>
      <phase>install</phase>
      <goals>
        <goal>build</goal>
      </goals>
    </execution>
  </executions>
</plugin>
{% endhighlight %}

If we run mvn clean install we will see how our image has been created successfully.

~~~ shell
mvn clean install
~~~

![Build Successfully]({{ "/img/Buildimange.png" | absolute_url }})

### How to connect our app to mongo db

First of all, we need to  add application.yml file. This file contains all configuration needed for our application

{% highlight yml %}
spring.data.mongodb:
   database: customers # Database name.
   uri: mongodb://mongo:27017 # Mongo database URI. Cannot be set with host, port and credentials.
{% endhighlight %}

One of the first questions that would fit me would be:
*Why does mongo appear as a host instead of localhost?*

For one reason:

If we put localhost and run our application with docker, the mongo database won’t be founded. Mongo will be located in a container and our app will be located in a different container.

However, If we run our application with java, we only have to add following line to our /etc/hosts file.

~~~ shell
mongo           127.0.0.1
~~~

Once the connection to mongo is configured, we need to add a repository to allow query database.

{% highlight java %}
import org.davromalc.tutorial.model.Customer;
import org.springframework.data.mongodb.repository.MongoRepository;
 
public interface CustomerRepository extends MongoRepository<Customer, String> {
 
}
{% endhighlight %}

Also, we need to enable mongo repositories in our main spring boot class.

{% highlight java %}
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.mongodb.repository.config.EnableMongoRepositories;
 
@EnableMongoRepositories
@SpringBootApplication
public class Application {
   public static void main(String[] args) {
      SpringApplication.run(Application.class, args);
   }
}
{% endhighlight %}

Additionally, we can add a controller that returns all the data, in json format, from mongo or to be able to persist new data.

{% highlight java %}
import java.util.List;
 
import org.davromalc.tutorial.model.Customer;
import org.davromalc.tutorial.repository.CustomerRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
 
import lombok.extern.slf4j.Slf4j;
 
@RestController
@Slf4j
public class CustomerRestController {
 
   @Autowired
   private CustomerRepository repository;
 
   @RequestMapping("customer/")
   public List findAll(){
      final List customers = repository.findAll();
      log.info("Fetching customers from database {}" , customers);
      return customers;
   }

   @RequestMapping(value = "customer/" , method = RequestMethod.POST)
   public void save(@RequestBody Customer customer){
      log.info("Storing customer in database {}", customer);
      repository.save(customer);
   }
}
{% endhighlight %}


Our application is fine. We only have to create a docker image to be able to run a container with our application.

~~~ shell
mvn clean install
~~~

Once the docker image is created, we can list all docker images and check if the docker image was created successfully.


![Docker Images]({{ "/img/dockerimagecreatedsuccessfully2.png" | absolute_url }})


Besides, we can run a container from the image created

![App Runs]({{ "/img/appsuccessfullyrunned.png" | absolute_url }})

At this point, we have our application dockerized but it cannot connect to mongo and we can’t see mongo data.



Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
