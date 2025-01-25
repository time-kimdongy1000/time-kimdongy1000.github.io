---
title: Spring MVC ExcelDown
author: kimdongy1000
date: 2023-04-21 09:20
categories: [Back-end, Spring - MVC]
tags: [ MVC , Excel ]
math: true
mermaid: true
---

## ExcelDown 
java는 엑셀을 포함해서 world의 일부분을 개발할 수 있는 라이브러리를 지원하는데 그것이 poi 라이브러리다 이 라이브러리를 활용하면 Excel 다운로드를 개발을 할 수 있는데
오늘은 그 Excel 다운로드에 대해서 알아보도록 하자

## HSSFWorkbook , XSSFWorkbook
java로 엑셀 다운로드를 만들다 보면 늘 어떤 라이브러리를 써야 하는지 헷갈릴 때가 있다 상황마다 다르지만 만들려는 파일의 확장자가 xlsx , xls 인지에 따라서 달라진다

기본적으로
xls는 낮은 버전의 엑셀 호환 확장자이다 (97~2003) 이는 HSSFWorkbook 객체를 만들어서 사용해야 하고
xlsx는 높은 버전의 엑셀 호환 확장자이다 (2007 이상의) 생성된 파일과 호환된다 각각 자신의 상황과 엑셀 호환 버전을 확인하고 개발을 진행해야 한다 오늘은 XSSFWorkbook 이것으로만 개발을 진행할 것이다


## maven 세팅

```

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.7.1</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.demo</groupId>
	<artifactId>Spring_Restart</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>Spring_Restart</name>
	<description>Project_Amadeus</description>
	<properties>
		<java.version>1.8</java.version>
	</properties>
	<dependencies>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-thymeleaf -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-thymeleaf</artifactId>
		</dependency>


		<!-- https://mvnrepository.com/artifact/org.projectlombok/lombok -->
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<version>1.18.24</version>
			<scope>provided</scope>
		</dependency>

		<!-- https://mvnrepository.com/artifact/org.apache.poi/poi -->
		<dependency>
			<groupId>org.apache.poi</groupId>
			<artifactId>poi</artifactId>
			<version>5.2.3</version>
		</dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.poi/poi -->
		<dependency>
			<groupId>org.apache.poi</groupId>
			<artifactId>poi</artifactId>
			<version>5.2.3</version>
		</dependency>

		<!-- https://mvnrepository.com/artifact/org.apache.poi/poi-ooxml -->
		<dependency>
			<groupId>org.apache.poi</groupId>
			<artifactId>poi-ooxml</artifactId>
			<version>5.2.3</version>
		</dependency>

		
	</dependencies>



	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>




```
의존성은 크게 없이 엑셀을 포맷을 만질 수 있는 org.apache.poi 세팅을 하면 엑셀 포맷을 작업할 수 있다 그럼 간단한 핸들러를 만들어보자
참고로 poi 라이브러리는 HSSFWorkbook만 사용할 수 있고 poi-ooxml는 XSSFWorkbook를 사용할 수 있다


## 임시폴더에 엑셀 파일 생성뒤 다운로드 
```
@Controller
public class ExcelDownController {

    private static final String tempFilePage = "C:\\TempFile\\";

    @Autowired
    private ResourceLoader resourceLoader;

    @GetMapping("/")
    public String goPage(){

        return "excelDown";
    }

    @GetMapping("/downExcel")
    public ResponseEntity<?> downExcel()
    {
        Workbook workbook = null;
        FileOutputStream fileOutputStream = null;
        try{

            workbook = new XSSFWorkbook(); // WorkBook 만들기
            String tempExcelFIleName = tempFilePage + UUID.randomUUID().toString() + ".xlsx"; // 임시 이름 및 임시 저장폴더 만들기
            fileOutputStream = new FileOutputStream(tempExcelFIleName); // FileOutputStream 으로 생성위치 명시

            Sheet sheet = workbook.createSheet("테스트 시트"); // 시트만들기

            workbook.write(fileOutputStream); // 엑셀 파일 FileOutPutStream 으로 명시된 위치로 생성

            /*자원반납*/
            workbook.close();
            fileOutputStream.close();

            
            /*임시폴더에서 파일 가져오기 사직*/
            File tempFile = new File(tempExcelFIleName);
            Resource resource =  resourceLoader.getResource("file:" + tempFile.getPath());

            /*파일 다운로드시 이름 붙여주는 곳 한글이름은 URLEncoder 을 붙여주어야 합니다*/
            String orginFileName = "테스트 엑셀파일입니다.xlsx";
            orginFileName = URLEncoder.encode(orginFileName , "UTF-8").replaceAll("\\+" , "%20");


            /*실제 파일 다운로드 return 엑셀 다운로드 ContetType 유의*/
            return ResponseEntity.ok().header(HttpHeaders.CONTENT_DISPOSITION, "attachement; filename=\"" + orginFileName + "\"")
                    .header(HttpHeaders.CONTENT_LENGTH, String.valueOf(tempFile.length()))
                    .contentType(MediaType.valueOf("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")).body(resource);

        }catch(Exception e){

            if(workbook != null){try {workbook.close();} catch (IOException ex) {throw new RuntimeException(ex);}}
            if(fileOutputStream != null){try {fileOutputStream.close();} catch (IOException ex) {throw new RuntimeException(ex);}}

            throw new RuntimeException(e);
        }

    }
}



```

실제 임시 폴더에 파일 생성하고 자원 반납 뒤 기존해 했던 파일 다운로드이다 좀 복잡해 보이기 때문에 실제 엑셀 파일을 생성하는 부분은 서비스 로직으로 분리해서 던지는 것이 좋을듯하다


## 변경된 Controller
```
@Controller
public class ExcelDownController {


    @Autowired
    private ExcelDownService excelDownService;

    @Autowired
    private ResourceLoader resourceLoader;

    @GetMapping("/")
    public String goPage(){

        return "excelDown";
    }

    @GetMapping("/downExcel")
    public ResponseEntity<?> downExcel()
    {
        
        try{
            String tempExcelFIleName = excelDownService.getTempExcelFileName();

            /*임시폴더에서 파일 가져오기 사직*/
            File tempFile = new File(tempExcelFIleName);
            Resource resource =  resourceLoader.getResource("file:" + tempFile.getPath());

            /*파일 다운로드시 이름 붙여주는 곳 한글이름은 URLEncoder 을 붙여주어야 합니다*/
            String orginFileName = "테스트 엑셀파일입니다.xlsx";
            orginFileName = URLEncoder.encode(orginFileName , "UTF-8").replaceAll("\\+" , "%20");


            /*실제 파일 다운로드 return */
            return ResponseEntity.ok().header(HttpHeaders.CONTENT_DISPOSITION, "attachement; filename=\"" + orginFileName + "\"")
                    .header(HttpHeaders.CONTENT_LENGTH, String.valueOf(tempFile.length()))
                    .contentType(MediaType.valueOf("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")).body(resource);

        }catch(Exception e){
            throw new RuntimeException(e);
        }

    }
}


```

## ExcelDownService

```
@Service
public class ExcelDownService {

    private static final String tempFilePage = "C:\\TempFile\\";


    public String getTempExcelFileName() throws  Exception{




        Workbook workbook = new XSSFWorkbook(); // WorkBook 만들기
        String tempExcelFIleName = tempFilePage + UUID.randomUUID().toString() + ".xlsx"; // 임시 이름 및 임시 저장폴더 만들기
        FileOutputStream fileOutputStream  = new FileOutputStream(tempExcelFIleName); // FileOutputStream 으로 생성위치 명시

        Sheet sheet = workbook.createSheet("테스트 시트"); // 시트만들기

        workbook.write(fileOutputStream); // 엑셀 파일 FileOutPutStream 으로 명시된 위치로 생성

        /*자원반납*/
        workbook.close();
        fileOutputStream.close();

        return tempExcelFIleName;
    }
}


```

결국 우리는 Spring MVC 철학에 매달려야 한다 Controller 은 매핑과 더불어서 서비스 호출 및 클라이언트 return 을 맡아야 하고 대부분의 로직은 Service에서 진행을 해야 한다
자 그럼 진행을 해보자 참고로 이 글의 끝에는 임시 폴더 저장 없이 바로 다운로드하는 것도 해볼 것이다

앞으로 엑셀 값 및 스타일은 ExcelDownService 소스에서
`Sheet sheet = workbook.createSheet("테스트 시트"); ` 과 `workbook.write(fileOutputStream); ` 사이에 넣을 것이다 참고 바랍니다

## 첫번째 Row 생성 

```

int startRow = 0;

Row headerRow = sheet.createRow(startRow);
headerRow.createCell(0).setCellValue("순번");
headerRow.createCell(1).setCellValue("학생번호");
headerRow.createCell(2).setCellValue("학생이름");
headerRow.createCell(3).setCellValue("교실명");
headerRow.createCell(4).setCellValue("담임 선생님");

```

java에서 엑셀을 다룰 때에는 모두 행렬 주소를 따라야 합니다 행 (가로) 열 (세로) 이들은 각각 주소를 가지고 있으며 0 이상의 정수로만 표현이 가능합니다
예를 들어서 지금처럼 startRow = 0으로 하고 Row headerRow = sheet.createRow(startRow);를 하게 되면 행의 0번 Row를 생성하겠다는 뜻입니다
이때 Row 0번은 엑셀 행 1번을 뜻합니다 그리고 createCell(0) 을 통하는데 이때 항상 headerRow를 참조하게 되는데 0번 Row의 몇 번째 열을 지목하게 되는 것이다
예를 들어서 지금 코드에서 순서대로 1A, 1B, 1C, 1D, 1E 위치를 뜻하고 그 위치에 해당 값을 넣어달라는 것입니다

## 리펙토링 

```

private void createRowAndCellData(Sheet sheet , int row , int col , String data){

    Row isCreatedRow = sheet.getRow(row);
    Row headerRow = null;

    if(isCreatedRow == null ){
        headerRow = sheet.createRow(row);
    }else{
        headerRow = isCreatedRow;
    }

    headerRow.createCell(col).setCellValue(data);

}

```

몇 줄 써놓고 무슨 리렉토링이냐고 할 수 있지만 엑셀 관련한 세팅은 데이터는 생각보다 너무 많고 데이터를 세팅할 때마다 row , col 하고 세팅을 해야 하므로 보통 하나의 메서드로 관리하는 게 편하다
이게 정답은 아니지만 나 같은 경우는 이렇게 해서 하나의 cell 을 컨트롤하는 편이다
메서드에 row를 생성할 sheet 와 생성할 각 위치 (row , col) 그리고 데이터이다 getRow로 현재 row에 row 가 생성되었는지 먼저 확인을 하고 있으면 생성하고 없으면 기존 것을 그대로 반환해서 사용한다
그리고 그 생성된 row에 col 위치를 명시해서 data를 입력하면 된다 그럼 호출할 때는 다음처럼 될 테인데


## 리펙토링 후 
```

/*첫번째 Row 만들기*/
int startRow = 0;
createRowAndCellData(sheet , startRow , 0 , "순번");
createRowAndCellData(sheet , startRow , 1 , "학생번호");
createRowAndCellData(sheet , startRow , 2 , "학생이름");
createRowAndCellData(sheet , startRow , 3 , "교실명");
createRowAndCellData(sheet , startRow , 4 , "담임선생님");

/*기존*/

int startRow = 0;

Row headerRow = sheet.createRow(startRow);
headerRow.createCell(0).setCellValue("순번");
headerRow.createCell(1).setCellValue("학생번호");
headerRow.createCell(2).setCellValue("학생이름");
headerRow.createCell(3).setCellValue("교실명");
headerRow.createCell(4).setCellValue("담임 선생님");


```

## 헤더에 스타일 넣기 

```

setCellStyleCustom(sheet , startRow , 0 , (byte)180 , (byte)50 , (byte)10);
setCellStyleCustom(sheet , startRow , 1 , (byte)100 , (byte)70 , (byte)20);
setCellStyleCustom(sheet , startRow , 2 , (byte)150 , (byte)90 , (byte)30);
setCellStyleCustom(sheet , startRow , 3 , (byte)200 , (byte)110 , (byte)40);
setCellStyleCustom(sheet , startRow , 4 , (byte)255 , (byte)130 , (byte)50);


private void setCellStyleCustom(Sheet sheet , int row , int col , byte r , byte g , byte b){

    Workbook workbook = sheet.getWorkbook();
    XSSFCellStyle cellStyle = (XSSFCellStyle)workbook.createCellStyle();
    cellStyle.setFillForegroundColor(new XSSFColor(new byte[] {(byte) r,(byte) g, (byte) b}, null)); //RGB 넣을려면 XSSFCellStyle 타입으로 생성을 해야 하며
    cellStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND); // 셀 지정된 색깔을 입힐려면 이 setFillPattern 함수를 호출해야하 한다 

    Row isCreatedRow = sheet.getRow(row);


    if(isCreatedRow == null ){
        throw new RuntimeException("Row 를 먼저 생성하고 진행해주세요");
    }

    Cell cell =  isCreatedRow.getCell(col);
    cell.setCellStyle(cellStyle);
}



```

보통 헤더에는 헤더라는 것을 강조하기 위해서 스타일을 조금 섞는 경우가 많다 헤더에 글자 크기 및 셀에 색깔을 넣어보자 마찬가지로 리팩토링이 진행된 채로 진행된 소스이며 이때 RGB는 byte 타입이며 동일한 RGB 값을 넣어서 사용할 수 있게 된다

## 폰트 및 글자 굵기 

```

setFontCustom(sheet , startRow , 0 , (String) null, false , (IndexedColors) null, (short) 1000);
setFontCustom(sheet , startRow , 1 , (String) "굴림", true , (IndexedColors) IndexedColors.BLACK , (short) 500);
setFontCustom(sheet , startRow , 2 , (String) "궁서", true , (IndexedColors) IndexedColors.BLUE , (short) 300);
setFontCustom(sheet , startRow , 3 , (String) "바탕", false , (IndexedColors) IndexedColors.AUTOMATIC , (short) 400);
setFontCustom(sheet , startRow , 4 , (String) "나눔고딕", true , (IndexedColors) IndexedColors.BROWN , (short) 600);

private void setFontCustom(Sheet sheet , int row , int col , String fontName , boolean flagBold  , IndexedColors fontColor , short fontHeight ){

    Workbook workbook = sheet.getWorkbook();

    Font font = workbook.createFont();

    Row isCreatedRow = sheet.getRow(row);
    Cell cell =  isCreatedRow.getCell(col);

    CellStyle cellStyle = cell.getCellStyle();


    if(isCreatedRow == null){
        throw new RuntimeException("Row 를 먼저 생성하고 진행해주세요");
    }


    if (fontName != null) {
        font.setFontName(fontName);
    } else {
        font.setFontName("나눔고딕");
    }

    font.setBold(flagBold);

    if(fontColor != null){
        font.setColor(fontColor.getIndex());
    }else{
        font.setColor(IndexedColors.BLACK.getIndex());
    }

    font.setFontHeight(fontHeight);
    
    cellStyle.setFont(font);
    cell.setCellStyle(cellStyle);
}

```

셀 색깔부터 글자 폰트 같은 것은 전부 CellStyle로 설정할 수 있다 이때 중요한 것은 스타의 객체를 한 곳에서 생성 다 면 다른 곳의 스타일은 새롭게 만들면 이전 스타일은 없어지게 되는데

이게 무슨 말인지 한번 보자 우리는 최초 셀에 색깔을 넣을 때 `XSSFCellStyle cellStyle = (XSSFCellStyle) workbook.createCellStyle();` 이 workbook에 대한 객체 값을 이미 만들어서 넣었다

다만 폰트 구현하는 곳에서 또 `XSSFCellStyle cellStyle = (XSSFCellStyle) workbook.createCellStyle();` 호출하게 되면 기존에 셀 스타일이 없어지게 된다

그래서 순서는 `XSSFCellStyle cellStyle = (XSSFCellStyle) workbook.createCellStyle();` 1번으로 오고 다음에 스타일을 수정하는 곳에서는

`CellStyle cellStyle = cell.getCellStyle();` 이렇게 가야 한다  그래서 순서는 먼저 createCellStyle() 호출로 CellStyle 만들고 그다음부터는 Cell에서 이미 구현되어 있는 것으로 불러오면 된다


## Merge 

```

startRow++;

createRowAndCellData(sheet , startRow , 0 , "데이터1");
createRowAndCellData(sheet , startRow , 2 , "데이터2");
createRowAndCellData(sheet , startRow , 4 , "데이터3");
createRowAndCellData(sheet , startRow , 6 , "데이터4");

cellMergeCustom(sheet , 1 , 1 , 0 , 1);
cellMergeCustom(sheet , 1 , 2 , 2 , 2);
cellMergeCustom(sheet , 1 , 3 , 4 , 5);

private void cellMergeCustom (Sheet sheet , int startRow , int endRow , int startCol , int endCol){
        sheet.addMergedRegion(new CellRangeAddress(startRow , endRow , startCol , endCol));
}



```
마지막으로 Cell 머지가 있다 CellMerge는 기본적인 바탕에 셀을 어디서부터 어디까지 머지 할까 하는 셀 컨트롤 일부이다 머지는 의외로 간단하다 Row 별로 시작, 끝, Col 별로 시작, 끝 지정하면
머지가 된다 이때 데이터가 들어가는 곳은 은 것은 최초 입력한 위치에서 입력되고 머지가 되는 형태가 될 것이다

이렇게 엑셀을 다양하게 활용하면 재미있게 할 수 있을 것이다 끝으로 이 엑셀은 처음에 임시파일을 만들고 그 임시파일을 내 보내는 형태이다 임시파일 만들지 않고 바로 만들어서 내보는 방법을 끝으로 종료를 할 것이다

```

@GetMapping("/downExcel2")
public void downExcel2(HttpServletRequest request , HttpServletResponse httpServletResponse)
{

    try{
        Workbook workbook = excelDownService.getWorkBook();

        /*파일 다운로드시 이름 붙여주는 곳 한글이름은 URLEncoder 을 붙여주어야 합니다*/
        String orginFileName = "테스트 엑셀파일입니다.xlsx";
        orginFileName = URLEncoder.encode(orginFileName , "UTF-8").replaceAll("\\+" , "%20");

        httpServletResponse.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
        httpServletResponse.setHeader(HttpHeaders.CONTENT_DISPOSITION, "attachement; filename=\"" + orginFileName + "\"");

        workbook.write(httpServletResponse.getOutputStream());
        workbook.close();


    }catch(Exception e){
        throw new RuntimeException(e);
    }

}

```
핸들러를 이렇게 작성하고 마지막으로 내보낼 때에는 workbook 상태에 httpServletResponse.getOutputStream() 보내시면 됩니다 어느 것이 더 편한지는 상황에 맞게 사용하시면 됩니다











