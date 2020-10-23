
'''
Sub column_list()
'pivot fejlécek listába másolása
Dim i As Integer

i = 5

Do While Range("A" & i) <> ""
    t = ""
    For Each c In Range("B" & i & ":CA" & i)
        If c.Value <> "" Then
        t = t & Cells(4, c.Column) & ", "
        End If
    Next
    Range("CC" & i) = t
i = i + 1
Loop

End Sub
'''
