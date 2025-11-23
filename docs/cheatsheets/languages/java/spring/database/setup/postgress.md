spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/appdb
    username: appuser
    password: apppass
  jpa:
    hibernate:
      ddl-auto: update # or validate | update | create | create-drop as needed
    #show-sql: false
