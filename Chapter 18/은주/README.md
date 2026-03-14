# Chapter 18. Spring Data R2DBC

## 18.1. R2DBC 란?
- R2DBC (Reactive Relational Database Connectivity) : 관계형 데이터베이스에 리액티브 프로그래밍 API 를 제공하기 위한 개방형 사양
- R2DBC 탄생 전에는 리액티브 애플리케이션에서 관계형 DB를 사용할 경우, JDBC API 자체가 Blocking API 여서 완전한 Non-Blocking I/O 를 지원하는 게 불가능했었음

## 18.3. Spring Data R2DBC 설정
- 테이블 스키마 정의
  - Spring Data JPA 처럼 엔티티에 정의된 매핑 정보로 테이블을 자동 생성해주는 기능이 없으므로, 테이블 스크립트를 직접 작성해서 테이블 생성 필요
