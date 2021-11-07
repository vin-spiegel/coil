# coil

>네코랜드용 가벼운 코루틴 기반 협력 스레딩 모듈

## 사용방법

  ```lua
  coil = require "coil"
  ```

- 시작 부분에서 `coil.update()` 를 호출하고 `deltaTime` 을 제공해야 합니다.

  ```lua
  coil.update(deltaTime)
  ```

  *사용법 1:*

  ```lua
  -- onTick리스트에 coil.update를 바로 넣어주는 방법입니다
  Client.onTick.Add(coil.update)
  ```

  *사용법 2:*

  ```lua
  -- coil:start() 함수는 onTick리스트에 coil.update 함수를 등록한 뒤 다시 리턴해줍니다
  ---@type fun(deltatime) coil.update
  local start = coil:start()
  ```

- 코일은 각 협력 스레드를 `tasks` 테이블에서 참조합니다. `coil.add()`함수로 새로운 `task`를 생성합니다.

  ```lua
  -- coil.update()에 델타 타임 제공
  Client.onTick.Add(coil.update)

  -- 1초에 한번씩 hello 총 5번 출력하기
  coil.add(function()
    for i = 1, 5 do
      print("hello")
      coil.wait(1)
    end
  end)
  ```

### coil.wait()

- 이 함수는 업데이트 하기 전에 다른 이벤트를 기다리는 데 사용할 수 있습니다.
- `wait` 함수에 인수로 숫자를 사용하여 호출되면 해당 시간 (초) 만큼 기다립니다.
- `coil.callback()` 에 의해 생성된 콜백으로 `wait` 가 호출되면 해당 콜백 함수가 호출 될 때 까지 `yield` 상태가 됩니다.
- `wait()` 가 인수 없이 호출되면 바로 다음 프레임까지만 `yield` 됩니다. 이런 방법은 매 tick당 업데이트 되는 코드를 안전하게 관리할 수 있습니다

  ```lua
  local coil = require("lib/coil.init")

  --coil.update함수에 온틱 deltaTime 매개변수 전달
  Client.onTick.Add(coil.update)

  local task = {
      -- coil.callback 함수를 coil.wait함수에 매개변수로 전달하면 `task`를 호출합니다
      call = coil.callback()
  }

  function task.say()
      print("이 함수는 다른 이벤트를 기다리는 데 사용할 수 있습니다")
      coil.wait(2)
      print("wait 함수에 인수로 숫자를 사용하여 호출하면")
      coil.wait(2)
      print("해당 시간만큼 기다립니다")
      -- 여기서부터는 task.call 함수가 coil.wait함수에 task.call 콜백을 담어 호출해야만 실행됩니다
      coil.wait(task.call)
      print("call은 호출 되어야만 실행됩니다")
  end

  -- task 추가
  coil.add(task.say)

  -- 6 초뒤 call하기
  coil.add(
      function()
          coil.wait(6)
          task.call()
      end
  )
  ```

### coil.callback()
  
- `coil.callback()` 함수를 사용하여 특수 콜백을 생성할 수 있습니다.`coil.wait()` 함수에 `coil.callback()`이 전달되면 `tasks`에 담긴 콜백이 호출됩니다.

    ```lua
    -- call()함수가 호출될때마다 "Hello world" 출력하기
    local function myTask()
      call = coil.callback()
      coil.wait(call)
      print("Hello world")
    end
    coil.add(myTask)
    
    call() --> "Hello world" 출력
    ```

### coil.update(dt)

- 다음과 같이 간단한 방법으로 안전한 루프를 관리할 수 있습니다.

### coil.add(fn)
  
- 새로운 `task` 를 추가하면 작업이 `coil.update()`에 대한 다음 호출에서 매 프레임 실행되기 시작합니다.

    ```lua
    coil.add(
        function()
            while true do
                print("2초에 한번씩 실행됩니다")
                coil.wait(2)
            end
        end
    )

    local function loop()
      while true do
        print("매틱당 yield후 실행됩니다")
        coil.wait()
      end
    end

    myLoop = coil.add(loop)
  ```

- `task` 는 `:stop()` 메서드를 호출하여 언제든지 중지 및 제거할 수 있습니다.
이렇게 하려면 작업이 생성될 때 변수에 할당되어야 합니다.

  ```lua
  -- 새로운 task 만들기
  local t = coil.add(function()
    coil.wait(1)
    print("hello")
  end)

  -- coil:update() 함수로 다시 실행되기 전에 멈추기
  t:stop()
  ```

### 그룹

- `coil` 은 `task` 그룹을 생성하는 기능을 제공합니다. 그룹 메서드는 `coil.group()`을 호출하여 생성됩니다.

  ```lua
  local group = coil.group()
  ```

- 그룹이 생성되면 `coil` 객체와 독립적으로 작동하며 반드시 `group:update(deltatime) 메소드로 각 프레임을 업데이트 시켜야 합니다.

  ```lua
  Client.onTick.Add(function(dt)
    group:update(dt)
  end)
  ```

- 그룹에 작업을 추가하려면 그룹의 ':add()' 메서드를 사용해야 합니다.
  
  ```lua
  group:add(function()
    coil.wait(10)
    print("10 초가 지났습니다")
  end)
  ```

  > 그룹이 유용하게 쓰일수 있는 상황은 여러 세트가 있는 게임입니다. 게임 월드 객체에 영향을 미치고 한번에 일시 중지 해야 하는 작업에 유용합니다. 새로운 `update()` 메소드 호출로 이전 업데이트를 무시할 수도 있고, 그룹을 파괴 시킴으로써 그룹에 할당된 작업도 모두 파괴시킬 수 있습니다.

### License

  > This library is free software you can redistribute it and/or modify it under the terms of the MIT license. See [LICENSE](LICENSE) for details.

## 원본

- [coil](https://github.com/rxi/coil) - rxi
