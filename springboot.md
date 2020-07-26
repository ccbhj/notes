# spring boot

## springboot-cli command  
1. **spring run filename**  // run a app
2. **jar tvf target/*.jar**, **java -jar**    // run the project package in the ./target
3.**jar -jar *.jar --debug** // failure Analyzer
## mvn command  
1. **mvn denpendency:tree**    // print dependencies tree
2. **mvn spring-boot:run**     // run the springboot project
3. **mvn package**    // show the package needed in pom.xml

## spring annotations
  
##### This-is annotations
1. **@RestController**    // rest controller (return json)
2. **@SpringBootApplication** // locate the main application class, spring will scan every entity from here, **Entrance of the application**
3. **@Configuration**    // l allow to register extra beans in the context or import additional configuration classes
4. **@Component**    // this is component
  

##### do-something annotations
1. **@EnableAutoConfiguration** // enable auto configuration 
2. **@ImportResource**    // import XML configuration files
3. **@ComponentScan** // enable @Component scan on the package where the application is located
4. **@RequestMapping** // mapping for url and method
5. **@GetMapping** 
6. **@PostMapping** 