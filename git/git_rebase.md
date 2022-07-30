# Git Rebase

<br>

- Git에서 한 브랜치에서 다른 브랜치로 합치는 방법은 2가지(Merge, Rebase)가 있다.

<br>

## 📌 상황

<img width="650" alt="image" src="https://user-images.githubusercontent.com/69254943/181924638-8fa6fdc0-4d6a-4678-a499-8304e4212ffe.png">

<br>

## 📌 적용

<br>

### 1. ```C4```의 변경사항을 ```C3```에 적용하는 Rebase 과정
```bash
$ git checkout experiment
$ git rebase master
```

<img width="639" alt="image" src="https://user-images.githubusercontent.com/69254943/181924799-7c05193a-7b81-450e-a1b6-094e4dbbce26.png">

- 두 브랜치가 나뉘기 전인 공통 커밋으로 이동하고 나서 그 커밋부터 지금 checkout한 브랜치(여기서는 experiment)가 가리키는 커밋까지 diff를 만들어 어딘가에 임시 저장 해둔다.
- Rebase할 브랜치(위에서 experiment)가 합칠 브랜치(master)가 가리키는 커밋을 가리키게 하고 아까 저장해 놓았던 변경사항을 차례대로 적용한다.

<br>

### 2. master 브랜치를 Fast-forward 시키기
```bash
$ git checkout master
$ git merge experiment
```

<img width="638" alt="image" src="https://user-images.githubusercontent.com/69254943/181924945-9183675a-1918-4f14-8d68-a059b3af8c12.png">

<br>

## 📌 특징

- 일을 병렬로 동시에 진행해도 Rebase하고 나면 모든 작업이 차례대로 수행된 것처럼 보인다.
- Rebase는 보통 remote branch에 커밋 내역을 깔끔하게 적용하고 싶을 때 사용한다.
- Rebase와 Merge의 최종 결과물은 같고 커밋 히스토리만 다르다.
  - Rebase : 브랜치의 변경사항을 순서대로 다른 브랜치에 적용하며 합침
  - Merge : 두 브랜치의 최종 결과만을 가지고 합침

