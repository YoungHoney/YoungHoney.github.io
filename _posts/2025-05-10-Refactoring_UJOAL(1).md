---
layout: post
title: 리팩토링-우리조상알기(1)
date: 2025-05-10
categories: [OOP,Spring,Refactoring,AI]
tags: [Java,Spring,AI]
---

# 1 들어가며

최근 OOP관련 개발서적을 읽으며 "좋은 개발이란 뭔가" 에 관하여 공부하면서 실제로 적용하자는 생각이 들었다. 아예 새로운 프로젝트를 시작하는 것은 시간이 너무 많이 걸릴것 같아 리팩토링을 해보기로 했다.

"우리조상알기" 프로젝트는 2023년 9월~12월동안 진행한 캡스톤디자인(1) 프로젝트이다. 당시 프론트 1명, 백엔드는 나를 포함한 2명으로 진행했었다. Java Spring을 배우고 처음으로 활용한 프로젝트였고 당시 프로젝트를 어떻게든 빠르게 완성하는 것을 목표로 하다보니 코드의 Quality가 낮았는데 그만큼 리팩토링을 적용할 부분이 많을 것으로 예상하여 가장 먼저 선정했다.

# 2 프로젝트 구조(수정 전)

## 2-1 아키텍처 다이어그램
<p style="text-align: center;">
  <img src="https://github.com/user-attachments/assets/812510c4-f0a6-4a11-bf9f-4b4b94889fa6" alt="이미지 없음" width="800" height="500" />
</p>

프로젝트는 위와 같은 구조로 이루어져있다. 전반적으로 Java Spring, RESTful API기반의 백엔드 웹서버와 React로 이루어진 프론트엔드 구조이며, 사용자 요청이 들어오면 [한국민족문화대백과사전](https://encykorea.aks.ac.kr/) 그리고 [한국고전종합DB](https://db.itkc.or.kr/) 에서 OpenAPI를 사용하거나 웹크롤링을 통해 관련 정보를 가져온 후, MariaDB로 조회, 추가, 수정을 거쳐 적절히 중간 데이터를 만들고 AI API를 통해 정제 및 요약을 거쳐 응답을 다시 사용자에게 보여준다.

## 2-2 ERD

상단부
<p style="text-align: center;">
  <img src="https://github.com/user-attachments/assets/f6f41d36-0ba8-4c1e-8a8f-8ed2055d25dd" alt="이미지 없음" width="800" height="700" />
</p>

이 ERD 일부는 특정 인물(민족대백과사전-Person)에 대한 정보와 그 인물의 조선왕조실록-Silok 에서의 기록, 그리고 두 정보로 인해 생성되는 4가지 정보를 나타낸다. 다이어그램 속 테이블명의 괄호안에는 코드의 Entity Class의 이름이 들어갔다. MBTI-Mbti, 활동요약-Lifesummary, 개인사건-Privatehistory, 관직순서-Govsequence는 AI API를 거쳐 특정 양식, 특정 형식을 가지도록 얻어낸다.



<br>
하단부

<p style="text-align: center;">
  <img src="https://github.com/user-attachments/assets/15bee8b3-58e7-4ac5-8c69-3a1e355cda0c" alt="이미지 없음" width="1000" height="600" />
</p>


하단부에는 일종의 정적데이터와 유사한 테이블들이 있다. 인물-Person데이터는 기본적인 인물의 정보가 들어가고 능력치 통솔무력지력정치매력(통무지정매)는 적절한 판단 과정을 거쳐 배정된다. 조선시대사건-Oldevents는 위키백과 등을 참고하여 조선시대의 중요한 사건을 미리 저장해놓은 테이블이다. 옛관직-Oldgov, 현대관직-Moderngov는 각각 [공공데이터포털-한국학중앙연구원_조선조관직명정보](https://www.data.go.kr/data/15052751/fileData.do) 그리고 2023년 하반기 기준 정부의 관료와 각 지자체장을 기준으로 .csv파일을 미리 생성해두었다. 그리고 관직비교-Govmatch는 옛 관직과 오늘날의 공무원 등급이 9가지로 나뉜다는 점을 토대로 적당히 매칭하였다.


# 3 리팩토링

최근에 읽고있는 책 [자바/스프링 개발자를 위한 실용주의 프로그래밍](https://product.kyobobook.co.kr/detail/S000213447953)을 읽으며 몇몇 안티패턴에 대해 알게되었고 그 패턴들의 예시에서 느껴지는 "냄새"가 내 코드에서도 느껴지는 것을 확인했다. 

사용자가 "이름(한자)"를 입력하면 프론트엔드에서 POST로 컨트롤러에 도달한다. 이후 새로운 사람을 추가하는 "NewmanService.doNewmanSetting"을 호출한다. 이때 doNewmanSetting이 온갖 "냄새"를 풍기고 있었다. 

```java
@Transactional
public Long doNewmanSetting(String name) throws Exception { //홍길동(洪吉洞) 형식으로 입력
    String[] pediaInfo=krPediaApi.getKrpediaInfo(name);
    List<SilokDocument> silokInfo=silLokApi.SilokExtractor(name);
    String[] namesetting = getNameSetting(name);
    String surnameHangul=namesetting[1];
    String surnameHanja=namesetting[0];
    String clanHangul=pediaInfo[6].split("\\(")[0];
    Clan clan=new Clan();
    ClanId clanid=new ClanId();
    Krpedia krpedia=new Krpedia();
    Person person=new Person();

    List<Govsequence> govsequences=new ArrayList<>();
    List<Privatehistory> phistories=new ArrayList<>();
    Lifesummary lifesummary = new Lifesummary();
    Mbti mbti =new Mbti();
    //^^gpt 관련 4개 엔티티, Mbti, Lifesummary, PrivateHistory, Govsequence
    int base=surnameHangul.charAt(0)-'가';
    int choIdx = base / (21 * 28);
    char cho=CHO[choIdx];
    surnameHangul=surnameHangul.substring(0,1);
    surnameHanja=surnameHanja.substring(0,1);
    clanid.setClanHangul(clanHangul);
    clanid.setSurnameHanja(surnameHanja);
    clanid.setSurnameHangul(surnameHangul);
    clan.setClanid(clanid);
    clan.setCho(cho);

    if(personrepository.findClanByWholeName(clanHangul+surnameHangul)==null) {
        personrepository.saveClan(clan);
    }
    //^^ Clan 엔티티 설정 ^^
    person.setName(name);
    person.setClan(clan);
    person.setPersonpicture(pediaInfo[7]);
    person.setBirthyear(pediaInfo[0]); //처치필요
    person.setDeathyear(pediaInfo[1]);
    person.setJa(pediaInfo[8]);
    person.setHo(pediaInfo[9]);
    person.setSiho(pediaInfo[10]);

    personrepository.save(person); //krpedia보다 먼저 나와야함
    person.setKrpedia(krpedia); //person -> krpedia
    // Person 설정정

    krpedia.setName(name);
    krpedia.setBirthyear(pediaInfo[0]);
    krpedia.setDeathyear(pediaInfo[1]);
    krpedia.setClanHangul(clanHangul);
    krpedia.setSurnameHangul(surnameHangul);
    krpedia.setSurnameHanja(surnameHanja);

    krpedia.setPersonpicture("picture sdfsdf");

    krpedia.setDefinition(pediaInfo[3]);
    krpedia.setDescription(pediaInfo[4]);
    krpedia.setMaintext(pediaInfo[5]);
    krpedia.setPerson(person); //krpedia -> person

    krPediaRepository.save(krpedia);

    // KrPedia 설정정
    lifesummary.setKrpedia(krpedia);
    gptRepository.save(lifesummary);
    mbti.setKrpedia(krpedia);
    gptRepository.save(mbti);

    String govInfo="";

    String orig_govsequence="we";
    govInfo=krpedia.getDefinition()+krpedia.getDefinition()+krpedia.getMaintext();
    orig_govsequence=azureApi.getGovsequence(govInfo);

    int upsm_count=0;

    String[] gov_parts=orig_govsequence.split(",");
    List<String> govseq=new ArrayList<>();

    for(int i=0;i<gov_parts.length;i++) {
        if(i<5) { //최대 5개만
            String part=gov_parts[i];
            String[] splitPart=part.split(":");
            if(splitPart.length>1) {
                govseq.add(splitPart[1].trim());
                Govsequence temp_govseq=new Govsequence();
                temp_govseq.setKrpedia(krpedia);
                govsequences.add(temp_govseq);
                System.out.println("govseq.get(i) = " + govseq.get(i));
                gptRepository.save(temp_govseq);

            } else {
                govseq.add("없음");
                Govsequence temp_govseq=new Govsequence();
                temp_govseq.setKrpedia(krpedia);
                govsequences.add(temp_govseq);
                System.out.println("else govseq.get(i) = " + govseq.get(i));


                if(upsm_count==0) {
                    gptRepository.save(temp_govseq);
                    upsm_count=1;
                }

            }
        }

    }
    System.out.println("govseq.size() = " + govseq.size());

    for(int i=0;i<govseq.size();i++) {


            govsequences.get(i).setSequnce_num(i+1);
            System.out.println("sadf");

            if(govRepository.findOldgov(govseq.get(i))!=null) {

                govsequences.get(i).setOldgov(govRepository.findOldgov(govseq.get(i)));
                System.out.println("있어요"+ govsequences.get(i).getOldgov().getName());
            }

            else {
                Oldgov temp=new Oldgov();

                Govmatch tmatch=new Govmatch();

                tmatch.setModerngov(govRepository.findModerngov("현대미상"));
                tmatch.setOldgov(temp);

                temp.setName(govseq.get(i)+"(현대미상)");
                temp.setIswarrior(false);
                temp.setRank("종9품");
                temp.setGovmatches(null);

                if(govRepository.findOldgov(temp.getName())==null) {
                    govRepository.save(temp);
                    govRepository.save(tmatch);
                    govsequences.get(i).setOldgov(temp);

                }
               
            }
        }

    int numOfSilok=silokInfo.size();

    for(int i=0;i<numOfSilok;i++) {
        Silok silok=new Silok();
        silok.setEventyear(Integer.parseInt(silokInfo.get(i).getPublicationYear()));
        silok.setContents(silokInfo.get(i).getContent());
        silok.setP_id(personrepository.findPersonInDBByName(name).getId()); //silok->person
        silokRepository.save(silok);
    }
  //  ^^ gpt 4개중 첫번째, Govsequence 입력
    String orig_ls="";
    String orig_ls_food=krpedia.getDefinition()+krpedia.getDescription()+krpedia.getMaintext();
    for(int i=0;i<silokInfo.size();i++) {
        orig_ls_food+=silokInfo.get(i).getContent();
    }
    orig_ls=azureApi.getLifesummary(orig_ls_food,name);
    lifesummary.setContents(orig_ls);
    //^^ gpt 4개중 두번째, lifesummary 입력
    String orig_MBTI="";
    String orig_MBTI_food=krpedia.getDefinition()+krpedia.getDescription()+krpedia.getMaintext();
    for(int i=0;i<silokInfo.size();i++) {
        orig_MBTI_food+=silokInfo.get(i).getContent();
    }
    orig_MBTI=azureApi.getMBTI(orig_MBTI_food,name);

    String[] real_mbti=orig_MBTI.split("\\]");
    mbti.setContents(real_mbti[1]);
    mbti.setMbti(real_mbti[0].substring(real_mbti[0].length()-4,real_mbti[0].length()));

    // ^^ gpt 4개중 세번쨰, Mbti 입력
    String orig_phistory="asd";
    String orig_phistory_food=krpedia.getDefinition()+krpedia.getDescription()+krpedia.getMaintext();
    for(int i=0;i<silokInfo.size();i++) {
        orig_phistory_food+=silokInfo.get(i).getContent();
    }
    orig_phistory=azureApi.getPrivateHistory(orig_phistory_food,name);

    String[] ph_parts=orig_phistory.split("\\$");
    List<String> phistory_year=new ArrayList<>();
    List<String> phistory_content=new ArrayList<>();

    for(int i=0;i<ph_parts.length;i++) {
        if (i < 6) {
            String part = ph_parts[i];
            String[] splitPart = part.split(":");
            if (splitPart.length > 1) { //


                Privatehistory ph = new Privatehistory();
                String year_info = splitPart[0].substring(0, 4);

                Pattern pattern = Pattern.compile("\\d{4}");
                if (pattern.matcher(year_info).matches()) {
                    phistory_year.add(year_info);
                    System.out.println("p year : " + splitPart[0].substring(0, splitPart[0].length() - 1));
                    phistory_content.add(splitPart[1]);
                    System.out.println("p content : " + splitPart[1]);
                    ph.setKrpedia(krpedia);
                    gptRepository.save(ph);
                    phistories.add(ph);
                } else {
                    // year_info가 형식에 맞지 않으면, 이 부분에서 처리 (예: 로깅, 에러 메시지 등)
                    System.out.println("Invalid year format: " + year_info);
                }
            }
        }
    }
    for(int i=0;i<phistory_year.size();i++) {
        phistories.get(i).setEventyear(Integer.parseInt(phistory_year.get(i).trim()));
        phistories.get(i).setContents(phistory_content.get(i));
    }
    //^^ gpt클래스 4개중 네번째, privatehistory 입력
    //^^ gpt 4가지 클래스 넣기 ^^
    Long temp=personrepository.findPersonInDBByName(name).getId();
    Integer[] abilities=virtualService.getAbilityById(temp); //id 를 요구하므로 save 이후에 나와야 함 + govsequence가 저장된 이후에 필요하므로 뒤에 등장
    person.setTong(abilities[0]);
    person.setMu(abilities[1]);
    person.setJi(abilities[2]);
    person.setJung(abilities[3]);
    person.setMae(abilities[4]);

    Long result=personrepository.findPersonInDBByName(name).getId();
    return result;

}
```

## 3-1 NewmanService.doNewmanSetting의 안티패턴들

🧱 1. God Method

    설명: 하나의 메서드가 너무 많은 책임(Responsibility)을 가짐. 

    위치: doNewmanSetting 전체

    냄새: 실록 저장, Krpedia/Person/Clan 설정, GPT 처리, 능력치 계산까지 모든 처리를 이 메서드 혼자서 수행

📜 2. Transaction Script

    설명: 객체 간 협력 없이 순서대로 로직을 절차적으로 수행하는 패턴. 도메인 객체는 단순한 데이터 구조로 전락함.

    위치: person.setName(...), krpedia.setDefinition(...) 등 모든 엔티티의 필드 직접 설정

    냄새: 진짜 객체지향 방식이라면 Person.createFrom(pediaInfo) 식의 도메인 중심 생성자/팩토리가 필요함

🧩 3. Temporal Coupling

    설명: 코드가 특정 순서로 실행되지 않으면 예외 발생. 순서 의존성이 강함.

    위치:

        person.setKrpedia(krpedia)는 person이 먼저 저장된 후에만 의미 있음

        abilities = virtualService.getAbilityById(...)는 govsequence 저장 후에만 가능

    냄새: 순서가 꼬이면 NullPointerException, InvalidState 오류 발생 가능

👃 4. Feature Envy

    설명: 객체 외부에서 해당 객체의 내부 필드를 조작함. OOP 설계에서 책임을 잘못 나눴다는 의미.

    위치:

        person.setXxx(...), krpedia.setXxx(...), govsequences.get(i).setOldgov(...) 등

    냄새: 외부 서비스에서 내부 필드까지 직접 건드리는 건 객체 캡슐화 위반

🔢 5. Magic Index / Magic String

    설명: 배열이나 문자열에서 인덱스나 구분자에 의미가 담겨 있지만, 코드에서는 의미 없는 숫자나 문자로 하드코딩

    위치:

        pediaInfo[6].split("\\(")[0], split(":"), split("\\$"), CHO[choIdx] 등

    냄새: API 응답 구조가 바뀌면 코드 전체가 깨질 위험 있음

💥 6. Large Transaction

    설명: 트랜잭션 범위가 너무 넓어, 외부 API 실패나 작은 오류에도 전체 롤백 발생

    위치: @Transactional이 전체 메서드를 감싸고 있고, 그 안에서 krPediaApi, azureApi 호출

    냄새: 외부 시스템 문제로 인해 DB 작업까지 모두 무효화됨 → 성능 저하 및 롤백 트러블

🔁 7. Duplicate Code / 반복 로직

    설명: 비슷한 반복 구조가 여러 번 나옴 (특히 GPT 결과 파싱 및 저장)

    위치:  gov_parts 처리, silokInfo 반복, ph_parts 반복 등

    냄새: 반복 로직을 메서드로 분리하거나, 공통 유틸로 추출할 필요 있음

🔎 8. Primitive Obsession

    설명: 도메인 객체가 될 수 있는 값들을 String, int 등 기본형으로만 처리

    위치: String[] pediaInfo, String[] gov_parts, String[] ph_parts 등

    냄새: 구조적 의미를 갖는 데이터라면 PediaInfo, GovSequenceInfo, PrivateHistoryInfo 등 DTO/VO 클래스로 만들 필요가 있음



## 3-2 NewmanService.doNewmanSetting의 중간 개선 코드

``` java
    @Transactional
    public Long doNewmanSetting(String name) throws Exception {
        NameInfo info = NameInfo.parse(name); // ex) 홍길동(洪吉洞) → {hangul, hanja}
        String[] pediaInfo = krPediaApi.getKrpediaInfo(name);
        List<SilokDocument> siloks = silLokApi.SilokExtractor(name);
       
        // Clan 생성 및 저장
        String clanHangul = pediaInfo[6].split("\\(")[0];
        Clan clan = clanRepository.findById(info.toClanId(clanHangul))
                .orElseGet(() -> clanRepository.save(ClanFactory.create(clanHangul, info.hangul, info.hanja)));
      
        // Person 생성 및 저장
        Person person = PersonFactory.create(name, pediaInfo, clan);
        personrepository.save(person);
     
        // Krpedia 생성 및 저장
        Krpedia krpedia = KrpediaFactory.create(name, pediaInfo, person);
        krPediaRepository.save(krpedia);
       
        // GPT 결과 생성 및 저장
        gptResultService.generateGovsequence(krpedia);
        gptResultService.generateLifesummary(krpedia, siloks);
        gptResultService.generateMbti(krpedia, siloks);
        gptResultService.generatePrivateHistory(krpedia, siloks);
      
        // 실록 저장
        silokService.saveAll(person, siloks);

        // 능력치 계산 및 반영
        Integer[] abilities = virtualService.getAbilityById(person.getId());
        PersonFactory.applyAbilities(person, abilities);
        return person.getId();
    }

```

| 안티패턴 이름                     | 개선 여부    | 설명                                                                                                                                 |
| --------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **1. God Method**           | ⚠️ 부분 개선 | 전체 메서드 길이는 확 줄었지만, 여전히 다양한 책임(객체 생성 + 저장 + 외부 API 호출 + 로직 흐름)을 하나의 메서드에서 다루고 있음.<br>→ `ApplicationService`와 `UseCase`를 나누면 더 개선 가능 |
| **2. Transaction Script**   | ✅ 대부분 해결 | `Factory`, `Service`를 통해 객체 생성/로직 실행 책임이 잘 분산됨.<br>이전의 절차형 코드는 객체 협력 기반 구조로 바뀜                                                     |
| **3. Temporal Coupling**    | ⚠️ 일부 남음 | `person` → `krpedia` → `gptResult` 순서 의존성은 여전히 있음. 순서 틀리면 문제 생김<br>→ 테스트하기 어려운 점은 여전함                                              |
| **4. Feature Envy**         | ✅ 개선됨    | 외부 메서드에서 직접 `setXxx()` 하지 않고, `Factory`를 통해 객체가 **자기 필드를 알아서 설정**                                                                  |
| **5. Magic Index / String** | ❌ 여전히 존재 | `pediaInfo[6].split("\\(")[0]` 같은 **하드코딩된 배열 인덱스** 여전히 남아있음<br>→ `PediaInfo` 같은 DTO 도입 필요                                          |
| **6. Large Transaction**    | ❌ 그대로    | 여전히 `@Transactional` 안에서 `krPediaApi`, `silLokApi` 호출함 → 외부 API 실패가 DB 롤백 야기<br>→ 외부 API는 트랜잭션 바깥에서 호출하도록 리팩토링 필요                  |
| **7. Duplicate Code**       | ✅ 해소     | 반복 구조 (`govsequence`, `lifesummary`, `mbti`, `privatehistory`)를 각각 `gptResultService`로 위임하여 정리됨                                    |
| **8. Primitive Obsession**  | ⚠️ 일부 개선 | `NameInfo`, `ClanId` 등 일부는 잘 객체화되었으나 `String[] pediaInfo`는 여전히 raw 구조<br>→ `PediaInfo` 클래스로 포장 필요                                  |


우선 이 중간개선으로 doNewmanSetting의 가독성을 크게 개선하게 되었다. 아직 갈 길은 멀지만 하나하나씩 해보겠다.