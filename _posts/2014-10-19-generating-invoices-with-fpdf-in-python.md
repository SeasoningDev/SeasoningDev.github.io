---
layout: post
title: Generating Invoices with FPDF in Python 2.x
---
One of my web projects required me to generate an invoice after a customer purchased a product. I could not find any existing, lightweight solutions that fit my requirements. In addition, I was already using FPDF to generate another document for a different part of the website, so I decided to use this package to roll my own.

I found a script that (kind of) did what I wanted. The only problem was, it was written for the PHP version of FPDF. Below is my port (with some small adaptations) of this script to Python. Here is [an example invoice](https://github.com/SeasoningDev/SeasoningDev.github.io/raw/master/assets/docs/invoice_example.pdf).

Credit to Xavier Nicolay for [the original script](http://www.fpdf.org/en/script/script20.php "Original invoice generating php script")

{% highlight python %}
# -*- coding: UTF-8 -*-
import datetime
from fpdf.fpdf import FPDF
from fpdf.php import sprintf
from math import sqrt, pi, cos, sin
from django.utils.datastructures import SortedDict

class Product():
    """
    A generic class for representing products that will be charged
    in an invoice
    
    """
    
    def __init__(self, ref='', description='', unit_price=0, amount=0):
        self.ref = ref
        self.description = description
        self.unit_price = float(unit_price)
        self.amount = amount
    
    def total_net_price(self):
        return self.unit_price * self.amount
    
    @property
    def total_net_price_string(self):
        return '%.2f' % self.total_net_price()
    
    @property
    def unit_price_string(self):
        return '%.2f' % self.unit_price

 
class InvoicePDF(FPDF):
    """
    This class behaves like a regular FPDF, except it will generate an 
    invoice with the given information on the current page on calling the
    render() method
    
    """
    _columns = None
    _formats = {}
    _products = []
    _products_rendered = 0
    
    def __init__(self, company_name='', company_info=[], invoice_number=0, date=None,
                 client_ref='', client_company_name='', client_company_info=[], client_VAT_num='',
                 payment_method='', VAT=0.25, products=[], watermark=None):
        self.company_name = company_name
        self.company_info = company_info
        self.invoice_number = invoice_number
        if date is None:
            date = datetime.date.today()
        self.date = date
        
        self.client_ref = client_ref
        self.client_company_name = client_company_name
        self.client_company_info = client_company_info
        self.client_VAT_num = client_VAT_num
        
        self.payment_method = payment_method
        self.VAT = VAT
        self.products = products
        
        self.watermark = watermark
        ret = super(InvoicePDF, self).__init__()
    
        self.add_font('DejaVu', '', '/usr/share/fonts/TTF/DejaVuSansCondensed.ttf', uni=True)
        self.add_font('DejaVu', 'B', '/usr/share/fonts/TTF/DejaVuSansCondensed-Bold.ttf', uni=True)
        return ret
    
    def render(self):
        """
        Render an invoice on the next page of this PDF
        
        """
        self.add_page()
        
        self._add_company_header(self.company_name,
                               self.company_info)
        self._add_invoice_number(str(self.invoice_number))
        self._add_page_number(1);
        self._add_date(self.date.strftime('%d/%m/%Y'))
        self._add_client(self.client_ref)
        self._add_client_company(self.client_company_name,
                               self.client_company_info)
        self._add_client_VAT(self.client_VAT_num)
                               
        self._add_payment_method(self.payment_method)
        self._add_invoice_date(self.date.strftime('%d/%m/%Y'))
        
        cols = SortedDict((("REFERENCE", 23),
                           ("PRODUCT", 84),
                           ("AMOUNT", 22),
                           ("UNIT PRICE", 26),
                           ("TOTAL (excl. VAT)", 35)))
        self._add_cols(cols)
        cols = {"REFERENCE": "L",
                "PRODUCT": "L",
                "AMOUNT": "C",
                "UNIT PRICE": "R",
                "TOTAL (excl. VAT)": "R"}
        self._add_line_format(cols)
        self._col_to_field = {'REFERENCE': 'ref',
                              'PRODUCT': 'description',
                              'AMOUNT': 'amount',
                              'UNIT PRICE': 'unit_price_string',
                              'TOTAL (excl. VAT)': 'total_net_price_string'}
        
        total_price = 0
        for product in self.products:
            self._add_product(product)
            total_price += product.total_net_price()
        
        self._add_total_calc(total_price, self.VAT)
        
        if self.watermark is not None:
            self._add_watermark(self.watermark)
            
        return self
            
        
    def _arc(self, x1, y1, x2, y2, x3, y3):
        h = self.h
        self._out(sprintf('%.2F %.2F %.2F %.2F %.2F %.2F c ', x1*self.k, (h-y1)*self.k,
                            x2*self.k, (h-y2)*self.k, x3*self.k, (h-y3)*self.k))
    
    def _rounded_rect(self, x, y, w, h, r, style=''):
        k = self.k
        hp = self.h
        if (style == 'F'):
            op = 'f'
        elif (style == 'FD' or style == 'DF'):
            op = 'B'
        else:
            op = 'S'
            
        my_arc = 4/3 * (sqrt(2) - 1)
        self._out(sprintf('%.2F %.2F m',(x+r)*k,(hp-y)*k ))
        xc = x+w-r 
        yc = y+r
        self._out(sprintf('%.2F %.2F l', xc*k,(hp-y)*k ))
    
        self._arc(xc + r*my_arc, yc - r, xc + r, yc - r*my_arc, xc + r, yc)
        xc = x+w-r 
        yc = y+h-r
        self._out(sprintf('%.2F %.2F l',(x+w)*k,(hp-yc)*k))
        self._arc(xc + r, yc + r*my_arc, xc + r*my_arc, yc + r, xc, yc + r)
        xc = x+r 
        yc = y+h-r
        self._out(sprintf('%.2F %.2F l',xc*k,(hp-(y+h))*k))
        self._arc(xc - r*my_arc, yc + r, xc - r, yc + r*my_arc, xc - r, yc)
        xc = x+r 
        yc = y+r
        self._out(sprintf('%.2F %.2F l',(x)*k,(hp-yc)*k ))
        self._arc(xc - r, yc - r*my_arc, xc - r*my_arc, yc - r, xc, yc - r)
        self._out(op)
    
    def _rotate(self, angle, x=-1, y=-1):    
        if (x == -1):
            x = self.x
            
        if (y == -1):
            y =self.y
            
        if (self.angle != 0):
            self._out('Q')
            
        self.angle =angle
        if (angle != 0):
            angle *= pi/180
            c = cos(angle)
            s = sin(angle)
            cx = x*self.k
            cy = (self.h - y) * self.k
            self._out(sprintf('q %.5F %.5F %.5F %.5F %.2F %.2F cm 1 0 0 1 %.2F %.2F cm', 
                              c, s, -s, c, cx, cy, -cx, -cy))
    
    def _endpage(self):
        if (self.angle != 0):
            self.angle = 0
            self._out('Q')
        super(InvoicePDF, self)._endpage()
        
        
    
    
    
    def _add_company_header(self, name, extra_lines):    
        x1 = 10
        y1 = 8
        
        self.set_xy(x1, y1)
        self.set_font('DejaVu', 'B', 12)
        length = self.get_string_width(name)
        self.cell(length, 2, name)
        
        self.set_font('DejaVu', '', 10)
        for line in extra_lines:
            y1 += 4
            self.set_xy(x1, y1)
            length = self.get_string_width(line)
            self.cell(length, 4, line)
    
    def _add_invoice_number(self, num):
        r1  = self.w - 80
        r2  = r1 + 68
        y1  = 6
        y2  = y1 + 2
        
        text = ("Invoice n° : %s" % num).decode('UTF-8')    
        szfont = 12
        loop = 0
        
        while (loop == 0):
            self.set_font("DejaVu", "B", szfont)
            sz = self.get_string_width(text)
            if (r1+sz) > r2:
                szfont -= 1
            else:
                loop += 1
        
        self.set_line_width(0.1)
        self.set_fill_color(192)
        self._rounded_rect(r1, y1, (r2 - r1), y2, 2.5, 'DF')
        self.set_xy(r1+1, y1+2)
        self.cell(r2-r1 -1, 5, text, 0, 0, "C")
    
    def _add_date(self, date):
        r1  = self.w - 61
        r2  = r1 + 30
        y1  = 17
        y2  = y1 
        mid = y1 + (y2 / 2)
        self._rounded_rect(r1, y1, (r2 - r1), y2, 3.5, 'D')
        self.line(r1, mid, r2, mid)
        self.set_xy(r1 + (r2-r1)/2 - 5, y1+3 )
        self.set_font("DejaVu", "B", 10)
        self.cell(10, 5, "DATE", 0, 0, "C")
        self.set_xy(r1 + (r2-r1)/2 - 5, y1+9 )
        self.set_font("DejaVu", "", 10)
        self.cell(10, 5, date, 0, 0, "C")
    
    def _add_client(self, ref):
        r1  = self.w - 31
        r2  = r1 + 19
        y1  = 17
        y2  = y1
        mid = y1 + (y2 / 2)
        self._rounded_rect(r1, y1, (r2 - r1), y2, 3.5, 'D')
        self.line(r1, mid, r2, mid)
        self.set_xy(r1 + (r2-r1)/2 - 5, y1+3)
        self.set_font("DejaVu", "B", 10)
        self.cell(10, 5, "CLIENT", 0, 0, "C")
        self.set_xy(r1 + (r2-r1)/2 - 5, y1 + 9)
        self.set_font("DejaVu", "", 10)
        self.cell(10, 5, ref, 0, 0, "C")
    
    def _add_page_number(self, page_num):
        r1  = self.w - 80
        r2  = r1 + 19
        y1  = 17
        y2  = y1
        mid = y1 + (y2 / 2)
        self._rounded_rect(r1, y1, (r2 - r1), y2, 3.5, 'D')
        self.line(r1, mid, r2, mid)
        self.set_xy(r1 + (r2-r1)/2 - 5, y1+3)
        self.set_font("DejaVu", "B", 10)
        self.cell(10, 5, "PAGE", 0, 0, "C")
        self.set_xy(r1 + (r2-r1)/2 - 5, y1 + 9)
        self.set_font("DejaVu", "", 10)
        self.cell(10, 5, str(page_num), 0, 0, "C")
    
    def _add_client_company(self, company_name, extra_lines):
        r1     = 10
        y1     = 40
        self.set_xy(r1, y1)
        self.set_font('DejaVu', 'B', 10)
        text = 'INVOICE TO:'
        self.cell(self.get_string_width(text), 4, text)
        y1 += 4
        self.set_xy(r1, y1)
        self.set_font('DejaVu', '', 10)
        self.cell(self.get_string_width(company_name), 4, company_name)
        
        for line in extra_lines:
            y1 += 4
            self.set_xy(r1, y1)
            length = self.get_string_width(line)
            self.cell(length, 4, line)
        
    def _add_client_adress(self, adress_lines):
        r1     = self.w - 80
        y1     = 44
        self.set_xy(r1, y1)
        for line in adress_lines:
            self.set_xy(r1, y1)
            length = self.get_string_width(line)
            self.cell(length, 4, line)
            y1 += 4
            
    def _add_payment_method(self, method):
        r1  = 10
        r2  = r1 + 60
        y1  = 80
        y2  = y1+10
        mid = y1 + ((y2-y1) / 2)
        self._rounded_rect(r1, y1, (r2 - r1), (y2-y1), 2.5, 'D')
        self.line(r1, mid, r2, mid)
        self.set_xy(r1 + (r2-r1)/2 -5 , y1+1 )
        self.set_font("DejaVu", "B", 10)
        self.cell(10, 4, "PAYMENT METHOD", 0, 0, "C")
        self.set_xy(r1 + (r2-r1)/2 -5 , y1 + 5 )
        self.set_font("DejaVu", "", 10)
        self.cell(10, 5, method, 0, 0, "C")
    
    def _add_invoice_date(self, date):
        r1  = 80
        r2  = r1 + 40
        y1  = 80
        y2  = y1+10
        mid = y1 + ((y2-y1) / 2)
        self._rounded_rect(r1, y1, (r2 - r1), (y2-y1), 2.5, 'D')
        self.line(r1, mid, r2, mid)
        self.set_xy(r1 + (r2 - r1)/2 - 5 , y1+1 )
        self.set_font("DejaVu", "B", 10)
        self.cell(10, 4, "INVOICE DATE", 0, 0, "C")
        self.set_xy(r1 + (r2-r1)/2 - 5 , y1 + 5 )
        self.set_font("DejaVu", "", 10)
        self.cell(10, 5, date, 0, 0, "C")
    
    def _add_client_VAT(self, vat):
        self.set_font("DejaVu", "B", 10)
        r1  = self.w - 80
        r2  = r1 + 70
        y1  = 80
        y2  = y1+10
        mid = y1 + ((y2-y1) / 2)
        self._rounded_rect(r1, y1, (r2 - r1), (y2-y1), 2.5, 'D')
        self.line(r1, mid, r2, mid)
        self.set_xy(r1 + 16 , y1+1 )
        self.cell(40, 4, "VAT NUMBER", '', '', "C")
        self.set_font("DejaVu", "", 10)
        self.set_xy(r1 + 16 , y1+5 )
        self.cell(40, 5, vat, '', '', "C")
    
    def _add_cols(self, tab):    
        r1  = 10
        r2  = self.w - (r1 * 2) 
        y1  = 100
        y2  = self.h - 50 - y1
        self.set_xy(r1, y1 )
        self.rect(r1, y1, r2, y2, "D")
        self.line(r1, y1+6, r1+r2, y1+6)
        colX = r1
        
        self._columns = tab
        for lib, pos in tab.items():
            self.set_xy(colX, y1+2 )
            self.cell(pos, 1, lib, 0, 0, "C")
            colX += pos
            self.line(colX, y1, colX, y1+y2)
    
    def _add_line_format(self, tab):
        for lib, fm in tab.items():
            self._formats[lib] = fm
    
    def _add_product(self, product):
        prod_y = 109 + 4*self._products_rendered
        x = 10
        
        for column, pos in self._columns.items():
            text = str(getattr(product, self._col_to_field[column]))
            formtext = self._formats[column]
            
            self.set_xy(x, prod_y)
            self.multi_cell(pos-2, 4, text, 0, formtext)
            x += pos
    
    def _add_total_calc(self, total_net_price, vat):
        self.set_font('DejaVu', 'B', 10)
        
        x = self.w - 55
        y = self.h - 45
        
        self.set_xy(x, y)
        self.cell(10, 4, "Total Excl. VAT:", align='R')
        y += 6
        self.set_xy(x, y)
        self.cell(10, 4, "VAT (%.2f%%):" % (vat*100), align='R')
        y += 10
        self.set_xy(x, y)
        self.cell(10, 4, "Total Incl. VAT:", align='R')
        
        x = self.w - 22
        y = self.h - 45
        
        self.set_xy(x, y)
        self.cell(10, 4, "%.2f" % (total_net_price), align='R')
        self.set_xy(x-16, y)
        self.cell(10, 4, u'€', align='R')
        y += 6
        self.set_xy(x, y)
        self.cell(10, 4, "%.2f" % (total_net_price * vat), align='R')
        self.set_xy(x-16, y)
        self.cell(10, 4, u'€', align='R')
        y += 10
        self.set_xy(x, y)
        self.cell(10, 4, "%.2f" % (total_net_price * (1 + vat)), align='R')
        self.set_xy(x-16, y)
        self.cell(10, 4, u'€', align='R')    
               
    def _add_watermark(self, text):
        self.set_font('DejaVu', 'B', 100)
        self.set_text_color(203, 203, 203)
        s_w = self.get_string_width(text)/sqrt(2)
        
        x = (self.w - s_w)/2
        
        self.rotate(45, x, 190)
        self.text(x, 190, text)
        self.rotate(0)
        self.set_text_color(0, 0, 0)
    
    
def generate_invoice_pdf():
    products = [Product(ref='PR01', description='Some Product', 
                        unit_price=100, amount=1)]
    pdfgen = InvoicePDF(company_name='My Company', 
                        company_info=['Somestreet 1',
                                         '1000 Somecity',
                                         'Somecountry'], 
                        invoice_number='PREVIEW', 
                        date=None, 
                        client_ref='CLI1', 
                        client_company_name='Dummy', 
                        client_company_info=['Somestreet 1',
                                                '1000 Somecity',
                                                'Somecountry'], 
                        client_VAT_num='BEXX XXXX XXX', 
                        payment_method='Mastercard', 
                        VAT=0.25, 
                        products=products, 
                        watermark=None)
    
    return pdfgen.render()
{% endhighlight %}
