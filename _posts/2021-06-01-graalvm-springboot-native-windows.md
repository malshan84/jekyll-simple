---
layout: post
title: 'windows에서 spring boot native 사용(with native-image, graalvm)'
---

# Spring boot native 준비

springboot native를 사용하는 방법은 아래 링크에 아주 친절하게 설명되어 있다.

[Spring Native documentation](https://docs.spring.io/spring-native/docs/current/reference/htmlsingle/)

# devtools 종속성 제거

기존 pom.xml에 위 링크에 있는 설정들을 추가 후 package를 하면 아래와 같은 에러가 발생하는 경우가 있다.

`Failed to execute goal org.springframework.experimental:spring-aot-maven-plugin:0.9.0:test-generate (test-generate) on project pms: Build failed during Spring AOT test code generation: Could not generate spring.factories source code: Devtools is not supported yet, please remove the related dependency for now. -> [Help 1]`

Spring project를 자동으로 생성했다면 dependencies에 아래 종속성이 있을것이다.

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <scope>runtime</scope>
  <optional>true</optional>
</dependency>
```

위에 있는 devtools종속성을 주석처리나 삭제하면 pacakage가 정상적으로 수행될 것이다.

# windows에서 native-image 생성 준비 작업

windows에 graalvm을 설정하는 방법은 아래 링크에 잘 나와 있다.

[Installation on Windows Platforms](https://www.graalvm.org/docs/getting-started/windows/)

windows의 native-image는 Visual stuio의 cl컴파일러를 사용하므로 Microsoft Windows SDK가 반드시 필요하다. 위 링크의 마지막 섹션을 참고해서 SDK를 설치하자.

# windows에서 native-image 생성

설정 완료 후 `mvn -Pnative-image package`를 수행 하면 아래와 같은 에러 메시지가 나오는 경우가 있다.

```
Error: Default native-compiler executable 'cl.exe' not found via environment variable PATH
Error: To prevent native-toolchain checking provide command-line option -H:-CheckToolchain
Error: Use -H:+ReportExceptionStackTraces to print stacktrace of underlying exception
Error: Image build request failed with exit status 1
```

cl.exe를 PATH에서 찾을 수 없다는 에러다. cl.exe의 path를 잡아야 하는데 "windows 시작"에서 `x64 Native Tools Command Prompt`로 검색하면 cl.exe와 Microsoft Windows SDK를 사용하는데 필요한 모든 path가 미리 설정된 prompt(cmd)창을 찾을 수 있다. 해당 prompt에서 mvn을 수행해야 한다.

자 이번에는 아까와 같이 cl.exe관련된 에러는 없을 것이다. 하지만 아래와 같은 에러가 발생할 것이다.

```
Error: Native-image building on Windows currently only supports target architecture: AMD64 ((x64) unsupported)
Error: To prevent native-toolchain checking provide command-line option -H:-CheckToolchain
Error: Use -H:+ReportExceptionStackTraces to print stacktrace of underlying exception
Error: Image build request failed with exit status 1
```

지원하지 않는 architecture라는 에러가 출력된다. github에서 graalvm 소스코드의 architecture를 validation하는 코드를 찾아보다 아래와 같은 코드를 발견했다.

```
 protected CompilerInfo createCompilerInfo(Path compilerPath, Scanner outerScanner) {
            try (Scanner scanner = new Scanner(outerScanner.nextLine())) {
                String targetArch = null;
                /* For cl.exe the first line holds all necessary information */
                if (scanner.hasNext("\u7528\u4E8E")) {
                    /* Simplified-Chinese has targetArch first */
                    scanner.next();
                    targetArch = scanner.next();
                }
                /*
                 * Some cl.exe print "... Microsoft (R) C/C++ ... ##.##.#####" while others print
                 * "...C/C++ ... Microsoft (R) ... ##.##.#####".
                 */
                if (scanner.findInLine("Microsoft.*\\(R\\) C/C\\+\\+") == null &&
                                scanner.findInLine("C/C\\+\\+.*Microsoft.*\\(R\\)") == null) {
                    return null;
                }
```

아마도 cl.exe를 실행해서 나오는 정보를 parsing해서 architecture를 추측하는 모양이다. cl.exe를 한 번 수행해 보면 아래와 같은 정보가 출력 된다.

```
Microsoft (R) C/C++ 최적화 컴파일러 버전 19.28.29336(x64)
Copyright (c) Microsoft Corporation. All rights reserved.

사용법: cl [ option... ] filename... [ /link linkoption... ]
```

추측해보면 아마도 locale이 한국이고 한글로된 메시지를 파싱하다 에러가 발생한듯 싶다.
일단 compiler를 validation 하는 부분을 skip하면 정상적으로 동작한다.
pom.xml에 추가한 native-image 플러그인에 다음 옵션을 추가하자.

```
<buildArgs>-H:-CheckToolchain</buildArgs>
```

아래와 같이 추가하면 된다.

```
.....
<plugin>
  <groupId>org.graalvm.nativeimage</groupId>
  <artifactId>native-image-maven-plugin</artifactId>
  <version>21.1.0</version>
  <configuration>
    <mainClass>com.malshan.example<mainClass>
    <buildArgs>-H:-CheckToolchain</buildArgs>
  </configuration>
  <executions>
    <execution>
      <goals>
        <goal>native-image</goal>
      </goals>
      <phase>package</phase>
    </execution>
  </executions>
</plugin>
......

```

이제 다시 `mvn -Pnative-image package`를 실행해 보자.
BUILD SUCCESS 메시지가 출력 되었다면 원래 jar파일이 생성되던 폴더에 exe파일이 하나 생겼을것이다.
실행해보면 신셰계를 경험하게 될것이다. 로딩 없이 엄청 빠르게 수행된다.
