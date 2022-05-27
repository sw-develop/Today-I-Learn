# 빈번하게 조회되는 테이블에 대한 조회 성능 개선하는 방법

<br>

**간단 패턴화**

테이블을 매번 join해서 조회할 때 성능 이슈가 발생한다면, 반정규화 / Redis 캐시 적용 / 분산 캐시 방안을 추가로 적용해볼 수 있다.

## 📌 상황
- 진행한 업무 중 ‘약국 조회 API 구현’을 진행하게 되었다. 검색 조건에 해당하는 약국 데이터를 반환하는 것이었다.

![image](https://user-images.githubusercontent.com/69254943/179789544-5ab5d347-44f1-41c8-bec4-693e52b96d28.png)

- 관련 테이블은 위의 사진과 같이 구성되어있고, 조회 시 전제는 특정 제휴기관과 제휴된(중개 테이블에 데이터가 존재하는) 약국을 대상으로 검색 조건을 적용해 반환한다는 것이다.

### 기존 조회 코드 구성
```java
//예시
public Page<PharmApiSearchResDto> getPharmBySearch(PharmApiSearchReqDto reqDto) {
        AffiliatedInstitution institution = affiliatedInstitutionRepository.findByAffiliatedInstitutionCd(reqDto.getAffiliatedInstitutionCd())
                .orElseThrow(() -> new CommonBadRequestException("affiliatedInstitutionNotFound")); //제휴기관 조회(1)

        Page<PharmApiSearchResDtoVo> resDtoVos = findPharmacy(institution, reqDto); //제휴약국정보 조회(2)
        return resDtoVos.map(pasrdv -> PharmApiSearchResDto.builder()
                .id(pasrdv.getId())
                .pharmacyName(pasrdv.getPharmacyName())
                .address(pasrdv.getAddress())
                .phoneNo(pasrdv.getPhoneNo()).build());
}
```

- 처음에 코드를 작성했을 때 제휴기관 조회(1) → 조건에 따른 제휴약국정보 조회(2) 순서로 구성하여 총 2번의 쿼리가 실행되었고, 위의 사진에서 정규화된 테이블을 모두 join해서 검색 조건 처리를 하도록 했다.
- **개선점 및 발생 가능한 문제점 (팀장님께서 언급해주신 발생가능한 성능 이슈 상황)**
    - 2번의 쿼리를 1번의 쿼리로 줄일 수 있다. (1)과 (2)의 쿼리를 하나로 줄일 수 있다.
    - 해당 API는 분명 요청이 많은 API이므로, 요청시마다 테이블들을 join한다면, 조회 성능 이슈가 발생할 수 있다.

<br>

## 📌 개선

### ▶️ 방안1 - 반정규화 & 1번의 쿼리로 조회하기
- 회사에서는 방안1로 우선 개선을 하였다.

### 반정규화
- 테이블 간의 join을 줄이기 위해 약국 조회 시 필요한 약국 테이블의 컬럼들과 제휴기관 테이블의 컬럼을 제휴약국 테이블이 포함하도록 반정규화한다.
- 약국 테이블, 제휴기관 테이블과의 join을 없앰 → 제휴약국 테이블만으로 조회 가능하도록 수정
- 수정한 설계 소개

![image](https://user-images.githubusercontent.com/69254943/179790349-81b33469-b423-44e6-9e05-e3bfabf0ea4f.png)

- 참고
    - [dataonair - 반정규화와 성능](https://dataonair.or.kr/db-tech-reference/d-guide/sql/?mod=document&uid=333) → 반정규화 기법 중 ‘중복컬럼 추가’를 사용함

### 1번의 쿼리로 줄이기
```java
if(시도, 시군구) 
	 제휴기관 & 제휴약국 정보 조회 - 데이터를 한번에
else (위도,경도)
	 제휴기관 & 제휴약국 정보 조회 - 데이터를 한번에
else //일반조회 
	 제휴기관 & 제휴약국 정보 조회 - 데이터를 한번에
```

- 제휴기관 조회를 따로 먼저 하지 않고, 위와 같이 조건에 따른 분기를 먼저 수행해 제휴기관 조회와 제휴약국정보 조회를 한 번의 쿼리로 수행되도록 한다.
- 개선한 코드 소개

    ```java
    /*1번의 쿼리로 줄이기 - 시도 & 시군구 or 위도 & 경도 or 일반 검색 조회*/
        private Page<PharmApiSearchResDtoVo> findPharmacy(PharmApiSearchReqDto reqDto) {
            if (isCityValExist(reqDto)) {
                return affiliatedPharmacyRepository.findByCity(
                        reqDto.getAffiliatedInstitutionCd(), reqDto.getSido(), reqDto.getSigungu(), reqDto.getPharmacyName(),
                        PageRequest.ofExternalApi(reqDto.getPage(), reqDto.getSize(), maxSize, "pharmacyName,asc")
                );
            }
    
            if (isLocationValExist(reqDto)) {
                return affiliatedPharmacyRepository.findByLocation(
                        reqDto.getAffiliatedInstitutionCd(), reqDto.getLatitude(), reqDto.getLongitude(), reqDto.getPharmacyName(),
                        PageRequest.ofExternalApi(reqDto.getPage(), reqDto.getSize(), maxSize, null)
                );
            }
    
            return affiliatedPharmacyRepository.findByPharmacyNameAndAffiliatedInstitution(
                    reqDto.getAffiliatedInstitutionCd(), reqDto.getPharmacyName(),
                    PageRequest.ofExternalApi(reqDto.getPage(), reqDto.getSize(), maxSize, "pharmacyName,asc")
            );
        }
    ```

### ▶️ 방안2 - 방안1 + Redis 캐시 적용
- 개선 방안1로 수정했는데 성능 이슈가 생긴다면, Redis 캐시 사용을 고려해보자
- Redis는 자주 변경되지 않고 조회가 빈번한 대상을 캐시에 저장해 사용하는 것이므로, 약국 조회 시 가장 빈번하게 조회될 제휴약국 테이블(위에서 중개 테이블)을 Redis에 저장해두는 것이다.
- 스프링에서는 Redis를 쉽게 추가해 사용(cacheable, cacheput, cache 삭제 등)할 수 있으므로 해당 방안은 추후에 성능 이슈가 생겼을 때 수정해도 충분하다고 판단했다.

### ▶️ 방안3 - 방안2 + 분산 캐시
- 만약 개선 방안2까지 했는데도 성능 이슈가 생긴다면, 이번에는 분산 캐시로 처리하는 것을 고려해보자