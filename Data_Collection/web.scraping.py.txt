from bs4 import BeautifulSoup
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.chrome.options import Options
import urllib, math, time, os, pandas as pd

options = Options()
# options.add_argument("--headless=new") 
# options.add_argument('--window-size=974,1047')
# options.add_argument('--window-position=953,0')
options.add_experimental_option("detach", True)

search = input('검색어:')
goal_cnt = int(input('스크래핑 할 건수는 몇건입니까?: '))
page_cnt = math.ceil(goal_cnt / 10)  # 크롤링 할 전체 페이지 수 

now = time.localtime()
date_format = '%04d%02d%02d'%(now.tm_year, now.tm_mon, now.tm_mday)
f_dir = f'{os.getcwd()}\\output\\{search}여행기사_{goal_cnt}건_{date_format}'
os.makedirs(f_dir, exist_ok=True) # 디렉토리가 미리 존재해도 에러나지 않도록

URL = 'https://korean.visitkorea.or.kr/search/search_list.do?keyword='+search
driver = webdriver.Chrome(options=options)
driver.get(URL)
time.sleep(2)

# 여행기사 더보기 클릭
driver.find_element(By.CSS_SELECTOR, "#s_recommend > .more_view > a").click()
time.sleep(2)

contents_no = 1
title_list = []
contents_list = []
img_url_list = []

def page_work():
    global contents_no, goal_cnt, title_list, contents_list, img_url_list
    result = driver.find_elements(By.CSS_SELECTOR,'#search_result .tit>a')
    
    for i in range(0, len(result)):
        if contents_no <= goal_cnt :    
            
            # 페이지 변경으로인한 DOM객체 변경 문제로 소스 다시 추출하기
            result = driver.find_elements(By.CSS_SELECTOR,'#search_result .tit>a')
            title = result[i].text
            title_list.append(title)
            
            print(f'[{contents_no}] {title}') 
            
            result[i].send_keys(Keys.ENTER) # .click()은 에러 잘남    
            time.sleep(2)
            
            # 이미지 추출을 위해 미리 스크롤
            driver.execute_script("window.scrollTo(0, document.body.scrollHeight)")
            time.sleep(2)
            
            # 상세페이지 소스 추출
            html = driver.page_source
            html_dom = BeautifulSoup(html, 'lxml')
          
            img_tag_list = html_dom.select('.img_typeBox img')
            img_url_list = [item['src'] for item in img_tag_list]

            contents = driver.find_elements(By.CLASS_NAME, 'txt_p')
            contents_merge = ' '.join([item.text for item in contents])        
            contents_list.append(contents_merge)           
            
            driver.back()
            time.sleep(2)     
            contents_no += 1
            
        if contents_no % 50 == 0: # 5page 단위 도달 시 넥스트 버튼 클릭
            driver.find_element(By.CSS_SELECTOR, '.btn_next').click()
            # continue

        
def file_export():
    global contents_no
    
    df = pd.DataFrame({"제목":title_list, "내용":contents_list}, index=None)

    filename = f'{search}여행기사_{goal_cnt}건_{date_format}.csv'
    fileaddr = f_dir+'\\'+filename
    
    if not os.path.exists(fileaddr):
        df.to_csv(fileaddr, index= False, mode='w', encoding='utf-8-sig')
    else:
        df.to_csv(fileaddr, index= False, mode='a', encoding='utf-8-sig', header=False)
    
    print(f'====== 콘텐츠 {contents_no-1}개, {filename} 파일 저장 완료 ======')
    
    img_no = 0
    for src in img_url_list:
        # 다운로드  (주소, 파일이름)
        img_no += 1
        urllib.request.urlretrieve(src, f'{f_dir}\\{page_no}_{img_no}.jpg')
        
    print(f'====== 이미지 {img_no}개 저장 완료 ======')

today = time.localtime()
print('스크래핑 프로그램 실행')

for page_no in range(1, page_cnt+1):    
    print(f'====== {page_no} 페이지 스크래핑 시작 ======')
    page_work()
    print(f'====== {page_no} 페이지 콘텐츠 저장 중 ======')
    file_export()
    print(f'====== {page_no} 페이지 스크래핑 완료 ======')
    if page_no < page_cnt:
        next_button = driver.find_element(By.CSS_SELECTOR, f"a[id='{page_no+1}']")
        driver.execute_script("arguments[0].click();", next_button)
        time.sleep(2)
                   
print('스크래핑 프로그램 종료')
driver.close()