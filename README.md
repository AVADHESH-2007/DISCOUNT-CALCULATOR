import { useState, useRef } from 'react';
import { Button } from "/components/ui/button";
import { Input } from "/components/ui/input";
import { Checkbox } from "/components/ui/checkbox";
import { Card, CardContent, CardHeader, CardTitle } from "/components/ui/card";
import { Plus, Minus, Upload, Download, RotateCcw, Check, FileText } from "lucide-react";

interface InvoiceRow {
  id: number;
  productCode: string;
  description: string;
  invoiceNumber: string;
  invoiceDate: string;
  dueDate: string;
  quantity: string;
  unitPrice: string;
  invoiceAmount: string;
  paymentDocNo: string;
  paymentDate: string;
  paymentAmount: string;
  discountRate: string;
  daysDifference?: number | string;
  proportionateQuantity?: number;
  discountAmount?: number;
  finalPayment?: number;
  isSelected?: boolean;
  isSplit?: boolean;
  parentId?: number;
  note?: string;
}

export default function DiscountCalculator() {
  const [rows, setRows] = useState<InvoiceRow[]>([
    {
      id: 1,
      productCode: 'PRD-001',
      description: 'Product 1',
      invoiceNumber: 'INV-001',
      invoiceDate: '15-10-2023',
      dueDate: '30-10-2023',
      quantity: '10',
      unitPrice: '25.50',
      invoiceAmount: '255.00',
      paymentDocNo: 'PAY-001',
      paymentDate: '20-10-2023',
      paymentAmount: '300.00',
      discountRate: '2',
      isSelected: false
    }
  ]);

  const fileInputRef = useRef<HTMLInputElement>(null);
  const [hasCalculated, setHasCalculated] = useState(false);
  const [hasChanged, setHasChanged] = useState(false);

  // Helper functions for date parsing and formatting
  const parseDate = (dateString: string) => {
    if (!dateString) return null;
    const parts = dateString.split('-');
    if (parts.length === 3) {
      const day = parseInt(parts[0], 10);
      const month = parseInt(parts[1], 10) - 1;
      const year = parseInt(parts[2], 10);
      return new Date(year, month, day);
    }
    return null;
  };

  const formatDateForDisplay = (dateString: string) => {
    if (!dateString) return '';
    if (/^\d{2}-\d{2}-\d{4}$/.test(dateString)) return dateString;
    const date = parseDate(dateString);
    if (date && !isNaN(date.getTime())) {
      const day = String(date.getDate()).padStart(2, '0');
      const month = String(date.getMonth() + 1).padStart(2, '0');
      const year = date.getFullYear();
      return `${day}-${month}-${year}`;
    }
    return dateString;
  };

  // Calculate days difference (due date - payment date)
  const calculateDifferentialDays = (dueDate: string, paymentDate: string) => {
    const due = parseDate(dueDate);
    const payment = parseDate(paymentDate);
    if (!due || !payment) return "N.A.";
    const timeDiff = due.getTime() - payment.getTime();
    const daysDiff = Math.floor(timeDiff / (1000 * 60 * 60 * 24));
    return daysDiff > 0 ? daysDiff : "N.A.";
  };

  // Calculate discount amount
  const calculateDiscount = (amount: number, discountRate: number, daysDiff: number | string) => {
    if (typeof daysDiff !== 'number') return 0;
    return amount * (discountRate / 100);
  };

  // Core reconciliation logic implementing the exact requirements
  const reconcileInvoicesAndPayments = (invoices: InvoiceRow[], payments: InvoiceRow[]) => {
    const result: InvoiceRow[] = [];
    let paymentIndex = 0;
    let remainingPayment = 0;
    let currentPayment: InvoiceRow | null = null;

    for (let i = 0; i < invoices.length; i++) {
      const invoice = invoices[i];
      let invoiceAmount = parseFloat(invoice.invoiceAmount) || 0;
      const discountRate = parseFloat(invoice.discountRate) || 0;
      const dueDate = invoice.dueDate;

      while (invoiceAmount > 0 && (paymentIndex < payments.length || remainingPayment > 0)) {
        if (remainingPayment <= 0) {
          currentPayment = payments[paymentIndex];
          remainingPayment = parseFloat(currentPayment.paymentAmount) || 0;
          paymentIndex++;
        }

        if (!currentPayment) break;

        const paymentDate = currentPayment.paymentDate;
        const daysDiff = calculateDifferentialDays(dueDate, paymentDate);
        const adjustedAmount = Math.min(invoiceAmount, remainingPayment);

        // Case 1: Exact match
        if (invoiceAmount === remainingPayment) {
          const adjustedRow: InvoiceRow = {
            ...invoice,
            id: result.length + 1,
            invoiceAmount: adjustedAmount.toFixed(2),
            paymentAmount: adjustedAmount.toFixed(2),
            paymentDocNo: currentPayment.paymentDocNo,
            paymentDate: currentPayment.paymentDate,
            daysDifference: daysDiff,
            discountAmount: calculateDiscount(adjustedAmount, discountRate, daysDiff),
            note: 'Full Match'
          };
          adjustedRow.finalPayment = adjustedAmount - (adjustedRow.discountAmount || 0);
          result.push(adjustedRow);
          invoiceAmount = 0;
          remainingPayment = 0;
        }
        // Case 2: Invoice < Payment
        else if (invoiceAmount < remainingPayment) {
          // Adjusted invoice row
          const adjustedRow: InvoiceRow = {
            ...invoice,
            id: result.length + 1,
            invoiceAmount: invoiceAmount.toFixed(2),
            paymentAmount: invoiceAmount.toFixed(2),
            paymentDocNo: currentPayment.paymentDocNo,
            paymentDate: currentPayment.paymentDate,
            daysDifference: daysDiff,
            discountAmount: calculateDiscount(invoiceAmount, discountRate, daysDiff),
            note: 'Adjusted'
          };
          adjustedRow.finalPayment = invoiceAmount - (adjustedRow.discountAmount || 0);
          result.push(adjustedRow);

          // Reduce remaining payment
          remainingPayment -= invoiceAmount;
          invoiceAmount = 0;

          // Only show excess payment for the last iteration
          if (i === invoices.length - 1 && remainingPayment > 0) {
            const excessRow: InvoiceRow = {
              id: result.length + 1,
              productCode: '',
              description: '',
              invoiceNumber: '',
              invoiceDate: '',
              dueDate: '',
              quantity: '',
              unitPrice: '',
              invoiceAmount: '0.00',
              paymentDocNo: currentPayment.paymentDocNo,
              paymentDate: currentPayment.paymentDate,
              paymentAmount: remainingPayment.toFixed(2),
              discountRate: '0',
              daysDifference: undefined,
              discountAmount: undefined,
              finalPayment: undefined,
              note: 'Excess Payment',
              isSplit: true
            };
            result.push(excessRow);
            remainingPayment = 0;
          }
        }
        // Case 3: Invoice > Payment
        else {
          // Adjusted payment row
          const adjustedRow: InvoiceRow = {
            ...invoice,
            id: result.length + 1,
            invoiceAmount: remainingPayment.toFixed(2),
            paymentAmount: remainingPayment.toFixed(2),
            paymentDocNo: currentPayment.paymentDocNo,
            paymentDate: currentPayment.paymentDate,
            daysDifference: daysDiff,
            discountAmount: calculateDiscount(remainingPayment, discountRate, daysDiff),
            note: 'Partially Adjusted'
          };
          adjustedRow.finalPayment = remainingPayment - (adjustedRow.discountAmount || 0);
          result.push(adjustedRow);

          // Reduce invoice amount
          invoiceAmount -= remainingPayment;
          remainingPayment = 0;

          // Only show excess invoice for the last iteration
          if (i === invoices.length - 1 && invoiceAmount > 0) {
            const excessRow: InvoiceRow = {
              ...invoice,
              id: result.length + 1,
              invoiceAmount: invoiceAmount.toFixed(2),
              paymentDocNo: '',
              paymentDate: '',
              paymentAmount: '0.00',
              daysDifference: undefined,
              discountAmount: undefined,
              finalPayment: undefined,
              note: 'Unadjusted Invoice',
              isSplit: true
            };
            result.push(excessRow);
            invoiceAmount = 0;
          }
        }
      }
    }

    return result;
  };

  // Main calculation function
  const calculateDiscounts = () => {
    // Separate invoices and payments
    const invoices = rows.filter(row => parseFloat(row.invoiceAmount) > 0);
    const payments = rows.filter(row => parseFloat(row.paymentAmount) > 0);

    // Perform reconciliation
    const reconciledRows = reconcileInvoicesAndPayments(invoices, payments);

    // Calculate proportional quantities and format dates
    const finalRows = reconciledRows.map(row => {
      const invoiceAmount = parseFloat(row.invoiceAmount) || 0;
      const unitPrice = parseFloat(row.unitPrice) || 1;
      
      return {
        ...row,
        invoiceDate: formatDateForDisplay(row.invoiceDate),
        paymentDate: formatDateForDisplay(row.paymentDate),
        dueDate: formatDateForDisplay(row.dueDate),
        proportionateQuantity: invoiceAmount / unitPrice,
        finalPayment: invoiceAmount - (row.discountAmount || 0)
      };
    });

    setRows(finalRows);
    setHasCalculated(true);
    setHasChanged(false);
  };

  // Add new row
  const addRow = () => {
    const newId = rows.length > 0 ? Math.max(...rows.map(row => row.id)) + 1 : 1;
    setRows([...rows, {
      id: newId,
      productCode: '',
      description: '',
      invoiceNumber: '',
      invoiceDate: '',
      dueDate: '',
      quantity: '',
      unitPrice: '',
      invoiceAmount: '',
      paymentDocNo: '',
      paymentDate: '',
      paymentAmount: '',
      discountRate: '',
      isSelected: false
    }]);
    setHasChanged(true);
  };

  // Remove selected rows
  const removeSelectedRows = () => {
    const remainingRows = rows.filter(row => !row.isSelected);
    if (remainingRows.length === 0) {
      addRow();
    } else {
      setRows(remainingRows);
    }
    setHasChanged(true);
  };

  // Reset selected rows
  const resetSelectedRows = () => {
    setRows(rows.map(row => {
      if (row.isSelected) {
        return {
          ...row,
          productCode: '',
          description: '',
          invoiceNumber: '',
          invoiceDate: '',
          dueDate: '',
          quantity: '',
          unitPrice: '',
          invoiceAmount: '',
          paymentDocNo: '',
          paymentDate: '',
          paymentAmount: '',
          discountRate: '',
          daysDifference: undefined,
          proportionateQuantity: undefined,
          discountAmount: undefined,
          finalPayment: undefined,
          note: undefined,
          isSplit: false
        };
      }
      return row;
    }));
    setHasChanged(true);
  };

  // Reset all rows
  const resetAll = () => {
    setRows([{
      id: 1,
      productCode: '',
      description: '',
      invoiceNumber: '',
      invoiceDate: '',
      dueDate: '',
      quantity: '',
      unitPrice: '',
      invoiceAmount: '',
      paymentDocNo: '',
      paymentDate: '',
      paymentAmount: '',
      discountRate: '',
      isSelected: false
    }]);
    setHasCalculated(false);
    setHasChanged(false);
  };

  // Toggle row selection
  const toggleRowSelection = (id: number) => {
    setRows(rows.map(row => {
      if (row.id === id) {
        return { ...row, isSelected: !row.isSelected };
      }
      return row;
    }));
  };

  // Handle input changes
  const handleInputChange = (id: number, field: keyof InvoiceRow, value: string) => {
    setRows(rows.map(row => {
      if (row.id === id) {
        return { ...row, [field]: value };
      }
      return row;
    }));
    setHasChanged(true);
  };

  // Import CSV
  const handleImportCSV = (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      const content = e.target?.result as string;
      const lines = content.split('\n').filter(line => line.trim());
      if (lines.length < 2) return;

      const headers = lines[0].split(',').map(h => 
        h.replace(/"/g, '').trim().toLowerCase().replace(/\s+/g, '')
      );

      const newRows = lines.slice(1).filter(line => line.trim()).map((line, index) => {
        const values = line.split(',').map(v => v.replace(/"/g, '').trim());
        
        const row: any = { 
          id: index + 1, 
          isSelected: false 
        };

        headers.forEach((header, i) => {
          if (i < values.length && values[i]) {
            const value = values[i];
            switch (header) {
              case 'productcode': row.productCode = value; break;
              case 'description': row.description = value; break;
              case 'invoicenumber': row.invoiceNumber = value; break;
              case 'invoicedate': row.invoiceDate = formatDateForDisplay(value); break;
              case 'duedate': row.dueDate = formatDateForDisplay(value); break;
              case 'quantity': row.quantity = value; break;
              case 'unitprice': row.unitPrice = value; break;
              case 'invoiceamount': row.invoiceAmount = value; break;
              case 'paymentdocno': row.paymentDocNo = value; break;
              case 'paymentdate': row.paymentDate = formatDateForDisplay(value); break;
              case 'paymentamount': row.paymentAmount = value; break;
              case 'discountrate': row.discountRate = value; break;
            }
          }
        });

        return row as InvoiceRow;
      });

      if (newRows.length > 0) {
        setRows(newRows);
        setHasCalculated(false);
        setHasChanged(true);
      }
    };
    reader.readAsText(file);
  };

  // Export CSV
  const handleExportCSV = () => {
    const headers = [
      'Product Code', 'Description', 'Invoice Number', 'Invoice Date', 'Due Date',
      'Quantity', 'Unit Price', 'Invoice Amount', 'Payment Doc No', 'Payment Date',
      'Payment Amount', 'Discount Rate', 'Days Difference', 'Proportionate Quantity',
      'Discount Amount', 'Final Payment', 'Note'
    ];
    
    const csvContent = [
      headers.join(','),
      ...rows.map(row => [
        `"${row.productCode}"`,
        `"${row.description}"`,
        `"${row.invoiceNumber}"`,
        `"${formatDateForDisplay(row.invoiceDate)}"`,
        `"${formatDateForDisplay(row.dueDate)}"`,
        `"${row.quantity}"`,
        `"${row.unitPrice}"`,
        `"${row.invoiceAmount}"`,
        `"${row.paymentDocNo}"`,
        `"${formatDateForDisplay(row.paymentDate)}"`,
        `"${row.paymentAmount}"`,
        `"${row.discountRate}"`,
        `"${row.daysDifference ?? ''}"`,
        `"${row.proportionateQuantity?.toFixed(2) ?? ''}"`,
        `"${row.discountAmount?.toFixed(2) ?? ''}"`,
        `"${row.finalPayment?.toFixed(2) ?? ''}"`,
        `"${row.note ?? ''}"`
      ].join(','))
    ].join('\n');

    const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.setAttribute('download', 'discount_calculator_export.csv');
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  // Generate sample CSV
  const generateSampleCSV = () => {
    const sampleData = [
      ['"Product Code"', '"Description"', '"Invoice Number"', '"Invoice Date"', '"Due Date"', '"Quantity"', '"Unit Price"', '"Invoice Amount"', '"Payment Doc No"', '"Payment Date"', '"Payment Amount"', '"Discount Rate"'],
      ['"PRD-001"', '"Product 1"', '"INV-001"', '"15-10-2023"', '"30-10-2023"', '"10"', '"25.50"', '"255.00"', '"PAY-001"', '"20-10-2023"', '"300.00"', '"2"'],
      ['"PRD-002"', '"Product 2"', '"INV-002"', '"16-10-2023"', '"31-10-2023"', '"5"', '"40.00"', '"200.00"', '"PAY-002"', '"21-10-2023"', '"200.00"', '"1.5"'],
      ['"PRD-003"', '"Product 3"', '"INV-003"', '"17-10-2023"', '"01-11-2023"', '"8"', '"15.75"', '"126.00"', '"PAY-003"', '"22-10-2023"', '"150.00"', '"3"']
    ].join('\n');

    const blob = new Blob([sampleData], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement('a');
    link.href = url;
    link.setAttribute('download', 'discount_calculator_sample.csv');
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
  };

  return (
    <div className="min-h-screen w-screen bg-gray-50 p-4">
      <Card className="w-full h-full">
        <CardHeader>
          <CardTitle className="text-2xl font-bold">Discount Calculator</CardTitle>
        </CardHeader>
        <CardContent className="h-[calc(100vh-180px)] overflow-auto">
          <div className="mb-4 flex flex-wrap gap-2">
            <Button onClick={addRow}>
              <Plus className="mr-2 h-4 w-4" /> Add Row
            </Button>
            <Button onClick={calculateDiscounts} disabled={!hasChanged}>
              <Check className="mr-2 h-4 w-4" /> Calculate
            </Button>
            <Button variant="outline" onClick={removeSelectedRows}>
              <Minus className="mr-2 h-4 w-4" /> Remove Selected
            </Button>
            <Button variant="outline" onClick={resetSelectedRows}>
              <RotateCcw className="mr-2 h-4 w-4" /> Reset Selected
            </Button>
            <Button variant="outline" onClick={resetAll}>
              <RotateCcw className="mr-2 h-4 w-4" /> Reset All
            </Button>
            <Button variant="outline" onClick={() => fileInputRef.current?.click()}>
              <Upload className="mr-2 h-4 w-4" /> Import CSV
              <input
                type="file"
                ref={fileInputRef}
                onChange={handleImportCSV}
                accept=".csv"
                className="hidden"
              />
            </Button>
            <Button variant="outline" onClick={generateSampleCSV}>
              <FileText className="mr-2 h-4 w-4" /> Sample CSV
            </Button>
            <Button variant="outline" onClick={handleExportCSV} disabled={!hasCalculated}>
              <Download className="mr-2 h-4 w-4" /> Export CSV
            </Button>
          </div>

          <div className="overflow-x-auto">
            <table className="min-w-full border">
              <thead className="bg-gray-100 sticky top-0">
                <tr>
                  <th className="border p-2 w-10">
                    <Checkbox 
                      checked={rows.every(row => row.isSelected)}
                      onCheckedChange={() => {
                        const allSelected = rows.every(row => row.isSelected);
                        setRows(rows.map(row => ({ ...row, isSelected: !allSelected })));
                      }}
                    />
                  </th>
                  <th className="border p-2">#</th>
                  <th className="border p-2">Product Code</th>
                  <th className="border p-2">Description</th>
                  <th className="border p-2">Invoice #</th>
                  <th className="border p-2">Invoice Date</th>
                  <th className="border p-2">Due Date</th>
                  <th className="border p-2">Qty</th>
                  <th className="border p-2">Unit Price</th>
                  <th className="border p-2">Amount</th>
                  <th className="border p-2">Payment Doc</th>
                  <th className="border p-2">Payment Date</th>
                  <th className="border p-2">Payment Amt</th>
                  <th className="border p-2">Discount %</th>
                  <th className="border p-2">Days Diff</th>
                  <th className="border p-2">Prop Qty</th>
                  <th className="border p-2">Discount Amt</th>
                  <th className="border p-2">Final Payment</th>
                  <th className="border p-2">Note</th>
                </tr>
              </thead>
              <tbody>
                {rows.map((row, index) => (
                  <tr key={row.id} className={`${row.isSelected ? 'bg-blue-50' : ''} ${row.isSplit ? 'bg-gray-50' : ''}`}>
                    <td className="border p-2">
                      <Checkbox 
                        checked={row.isSelected}
                        onCheckedChange={() => toggleRowSelection(row.id)}
                      />
                    </td>
                    <td className="border p-2">{index + 1}</td>
                    <td className="border p-2">
                      <Input
                        value={row.productCode}
                        onChange={(e) => handleInputChange(row.id, 'productCode', e.target.value)}
                        className="min-w-[120px]"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        value={row.description}
                        onChange={(e) => handleInputChange(row.id, 'description', e.target.value)}
                        className="min-w-[150px]"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        value={row.invoiceNumber}
                        onChange={(e) => handleInputChange(row.id, 'invoiceNumber', e.target.value)}
                        className="min-w-[100px]"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        type="text"
                        value={row.invoiceDate}
                        onChange={(e) => handleInputChange(row.id, 'invoiceDate', e.target.value)}
                        placeholder="DD-MM-YYYY"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        type="text"
                        value={row.dueDate}
                        onChange={(e) => handleInputChange(row.id, 'dueDate', e.target.value)}
                        placeholder="DD-MM-YYYY"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        type="number"
                        value={row.quantity}
                        onChange={(e) => handleInputChange(row.id, 'quantity', e.target.value)}
                        className="w-20"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        type="number"
                        value={row.unitPrice}
                        onChange={(e) => handleInputChange(row.id, 'unitPrice', e.target.value)}
                        step="0.01"
                        className="w-24"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        type="number"
                        value={row.invoiceAmount}
                        onChange={(e) => handleInputChange(row.id, 'invoiceAmount', e.target.value)}
                        step="0.01"
                        className="w-24"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        value={row.paymentDocNo}
                        onChange={(e) => handleInputChange(row.id, 'paymentDocNo', e.target.value)}
                        className="min-w-[100px]"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        type="text"
                        value={row.paymentDate}
                        onChange={(e) => handleInputChange(row.id, 'paymentDate', e.target.value)}
                        placeholder="DD-MM-YYYY"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        type="number"
                        value={row.paymentAmount}
                        onChange={(e) => handleInputChange(row.id, 'paymentAmount', e.target.value)}
                        step="0.01"
                        className="w-24"
                      />
                    </td>
                    <td className="border p-2">
                      <Input
                        type="number"
                        value={row.discountRate}
                        onChange={(e) => handleInputChange(row.id, 'discountRate', e.target.value)}
                        step="0.01"
                        className="w-20"
                      />
                    </td>
                    <td className="border p-2 text-center">
                      {row.daysDifference ?? '-'}
                    </td>
                    <td className="border p-2 text-center">
                      {row.proportionateQuantity?.toFixed(2) ?? '-'}
                    </td>
                    <td className="border p-2 text-center">
                      {row.discountAmount?.toFixed(2) ?? '-'}
                    </td>
                    <td className="border p-2 text-center">
                      {row.finalPayment?.toFixed(2) ?? '-'}
                    </td>
                    <td className="border p-2 text-sm text-gray-600">
                      {row.note || ''}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
