# Web_Scrapping-RAM_Prices_analysis

Due to the increased need for Hardware for AI development, There is a shortage in hardware and prices went up drastically,
so it may be a bit tough to make a purchase. Thats excactly the goal of this project.

In this project we use python to scrap computer memory (RAM) product pages to collect info about the currently available
products using requests and beautiful soup and then we proccess the data using pandas, and finally we load the data
into Power BI to create the final dashboard to have an over view of the available products and determine the best value product.

## Code Overview

Ofcourse the first thing we do is import the libraries we gonna need

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time
import re
```

Then, before our main function, we need to identify two functions that extract the DDR type and the size of RAM out of the product name, so we can use these functions using .apply() method provided by pandas on the the dataframe.

```python
def get_ddr(ram):
    if ram.find("DDR") != -1:
        return ram[ram.find("DDR"):ram.find("DDR")+4]
    else:
        return "unspecified"

def get_gb(ram):
    ram_sizes = ["512GB", "256GB", "192GB", "128GB", "96GB", "64GB", "32GB", "24GB", "16GB", "12GB", "8GB", "4GB", "2GB"]
    clean_ram = ram.replace(" ","").replace("RGB","").upper()
    matches = [word for word in ram_sizes if word in clean_ram]
    return matches[0] if matches else "unspecified"
```

Now for the main function, which we will devide into a few sections, first we declare the variables with empty lists so the data is refreshed everytime the script runs, then we have the initial request, which is used to get the total number of pages available on the website:

```python
# Main function
def ram_scrapper():
    # using while true try and except for the whole function for recursion, so when one of the requests fail, the function restarts
    while True:
        try:
            # declaring data lists outside of main loop, so they only reset upon calling the function
            item_list = []
            price_list = []
            rating_list = []
            link_list = []
            img_link_list = []

            # this section purpose to get total number of pages in search result to loop on it
            headers = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.36'}
            url = "https://www.amazon.eg/s?i=electronics&rh=n%3A21833213031&s=popularity-rank&fs=true&language=en&ref=lp_21833213031_sar"
            initial_page = requests.get(url,headers=headers)
            initial_soup = BeautifulSoup(initial_page.text,"html.parser")
            max_pn = int(initial_soup.find("span",{"class":"s-pagination-item s-pagination-disabled"}).text)
```

And then we enter main loop, where it request each page and then we have 5 sub-loops to extract the needed data from each page.

```python
 # main page loop
            for pn in range(1,max_pn+1):
                # this delay timer is needed because Amazon blocks web scrapping if it detects too many requests
                time.sleep(5)
                url_n = f"https://www.amazon.eg/-/en/s?i=electronics&rh=n%3A21833213031&s=popularity-rank&fs=true&page={pn}&language=en&xpid=j0S4UWeQ-OYl0&qid=1766858584&ref=sr_pg_{pn}"
                page_n = requests.get(url_n,headers=headers)
                soup_n = BeautifulSoup(page_n.text,"html.parser")
                cards = soup_n.find_all("div",{"class":"a-section a-spacing-small puis-padding-left-small puis-padding-right-small"}) # soup of cards with title, price...etc
                images = soup_n.find_all("div",{"class","a-section aok-relative s-image-square-aspect"}) # soup of images above cards

                # first loop for item name
                for item in cards:
                        item_name = item.find("h2",{"class":"a-size-base-plus a-spacing-none a-color-base a-text-normal"}).text
                        item_list.append(item_name)

                # second loop for item price
                for price in cards:
                    if price.find("span",{"class":"a-offscreen"}):
                        price_list.append(price.find("span",{"class":"a-offscreen"}).text)
                    else:
                        try:
                            price_list.append(price.find("div",{"class":"a-row a-size-base a-color-secondary"}).find("span",{"class":"a-color-base"}).text)
                        except:
                            AttributeError
                            price_list.append("unavailable")

                # third loop for item rating
                for rate in cards:
                        if rate.find("div",{"class":"a-row a-size-small"}):
                            rating_list.append(rate.find("div",{"class":"a-row a-size-small"}).find("span",{"class":"a-size-small a-color-base"}).text)
                        else:
                            rating_list.append("no rating")

                # fourth loop for item link
                for link in cards:
                    item_link = link.find("a",{"class":"a-link-normal s-line-clamp-4 s-link-style a-text-normal"})["href"]
                    link_list.append(f"https://www.amazon.eg{item_link}")

                # fifth loop for image link
                for img in images:
                        img_link = img.find("img",{"class":"s-image"})["src"]
                        img_link_list.append(img_link)
```

And then after we get all the data, we create a pandas dataframe to hold the extracted data and we perform the needed cleaning on colums like price and rating and extract the categories using the functions we identified earlier

```python
            # now after main loop finishes, the code should append all pages into this data frame
            data = {"item_name":item_list,"item_price":price_list,"item_rating":rating_list,"item_link":link_list,"image_links":img_link_list}
            df = pd.DataFrame(data)

            # cleaninig the data and extracting needed info
            # dropping unavailable items
            df = df.drop(labels=df[["item_price"]][df["item_price"] == "unavailable"].index).reset_index(drop=True)
            # converting price column to float 
            df["item_price"] = df["item_price"].str.replace("EGP","").str.replace(",","").str.strip().astype("float")
            # dropping items below 400 egp because they are ic not ram
            df = df.drop(df[df["item_price"] < 400].index).reset_index(drop=True)
            # extracting ddr and gb size for further categorization
            df["DDR_type"] = df["item_name"].apply(get_ddr)
            df["GB_size"] = df["item_name"].apply(get_gb)
            df["GB_num"] = df["GB_size"].replace("[a-zA-Z]","",regex=True)

            df.to_csv("D:/DA projects/python tasks/task5/Output/amazon_ram_data.csv",index=False)
            return print("Date Saved Successfully.")
            # End of main function
```

And finally we have an except statement incase any error occured.

```python
        # except statements for errors.
        except:
             AttributeError
             print("Error occurred during a request, script will restart")
             return ram_scrapper()
```


You can find the full Code in the Scripts folder and the interactive Dashboard <a href="https://app.powerbi.com/view?r=eyJrIjoiZWIwYWMxZTctZWEwZS00NmY0LWI2ZjktYWY2YWViMTNmMzllIiwidCI6IjU5ZDRjODc4LTE4NTEtNDFkNC05ZmVmLTY5MzE2ODYyMjI5OCJ9">Here.</a>
