
QMAKE_CXX.QT_COMPILER_STDCXX = 199711L
QMAKE_CXX.QMAKE_MSC_VER = 1916
QMAKE_CXX.QMAKE_MSC_FULL_VER = 191627045
QMAKE_CXX.COMPILER_MACROS = \
    QT_COMPILER_STDCXX \
    QMAKE_MSC_VER \
    QMAKE_MSC_FULL_VER
QMAKE_CXX.INCDIRS = \
    "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Tools\\MSVC\\14.16.27023\\ATLMFC\\include" \
    "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Tools\\MSVC\\14.16.27023\\include" \
    "C:\\Program Files (x86)\\Windows Kits\\NETFXSDK\\4.6.1\\include\\um" \
    "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.20348.0\\ucrt" \
    "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.20348.0\\shared" \
    "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.20348.0\\um" \
    "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.20348.0\\winrt" \
    "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.20348.0\\cppwinrt"
QMAKE_CXX.LIBDIRS = \
    "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Tools\\MSVC\\14.16.27023\\ATLMFC\\lib\\x64" \
    "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Tools\\MSVC\\14.16.27023\\lib\\x64" \
    "C:\\Program Files (x86)\\Windows Kits\\NETFXSDK\\4.6.1\\lib\\um\\x64" \
    "C:\\Program Files (x86)\\Windows Kits\\10\\lib\\10.0.20348.0\\ucrt\\x64" \
    "C:\\Program Files (x86)\\Windows Kits\\10\\lib\\10.0.20348.0\\um\\x64"

/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#include "custominputhandler.h"

#include <QtDataVisualization/Q3DCamera>
#include <QtCore/qmath.h>

CustomInputHandler::CustomInputHandler(QAbstract3DGraph *graph, QObject *parent) :
    Q3DInputHandler(parent),
    m_highlight(0),
    m_mousePressed(false),
    m_state(StateNormal),
    m_axisX(0),
    m_axisZ(0),
    m_speedModifier(20.0f)
{
    // Connect to the item selection signal from graph
    connect(graph, &QAbstract3DGraph::selectedElementChanged, this,
            &CustomInputHandler::handleElementSelected);
}

void CustomInputHandler::mousePressEvent(QMouseEvent *event, const QPoint &mousePos)
{
    if (Qt::LeftButton == event->button()) {
        m_highlight->setVisible(false);
        m_mousePressed = true;
    }
    Q3DInputHandler::mousePressEvent(event, mousePos);
}

//! [1]
void CustomInputHandler::wheelEvent(QWheelEvent *event)
{
    float delta = float(event->angleDelta().y());

    m_axisXMinValue += delta;
    m_axisXMaxValue -= delta;
    m_axisZMinValue += delta;
    m_axisZMaxValue -= delta;
    checkConstraints();

    float y = (m_axisXMaxValue - m_axisXMinValue) * m_aspectRatio;

    m_axisX->setRange(m_axisXMinValue, m_axisXMaxValue);
    m_axisY->setRange(100.0f, y);
    m_axisZ->setRange(m_axisZMinValue, m_axisZMaxValue);
}
//! [1]

void CustomInputHandler::mouseMoveEvent(QMouseEvent *event, const QPoint &mousePos)
{
    // Check if we're trying to drag axis label
    if (m_mousePressed && m_state != StateNormal) {
        setPreviousInputPos(inputPosition());
        setInputPosition(mousePos);
        handleAxisDragging();
    } else {
        Q3DInputHandler::mouseMoveEvent(event, mousePos);
    }
}

void CustomInputHandler::mouseReleaseEvent(QMouseEvent *event, const QPoint &mousePos)
{
    Q3DInputHandler::mouseReleaseEvent(event, mousePos);
    m_mousePressed = false;
    m_state = StateNormal;
}

void CustomInputHandler::handleElementSelected(QAbstract3DGraph::ElementType type)
{
    switch (type) {
    case QAbstract3DGraph::ElementAxisXLabel:
        m_state = StateDraggingX;
        break;
    case QAbstract3DGraph::ElementAxisZLabel:
        m_state = StateDraggingZ;
        break;
    default:
        m_state = StateNormal;
        break;
    }
}

void CustomInputHandler::handleAxisDragging()
{
    float distance = 0.0f;

    // Get scene orientation from active camera
    float xRotation = scene()->activeCamera()->xRotation();

    // Calculate directional drag multipliers based on rotation
    float xMulX = qCos(qDegreesToRadians(xRotation));
    float xMulY = qSin(qDegreesToRadians(xRotation));
    float zMulX = qSin(qDegreesToRadians(xRotation));
    float zMulY = qCos(qDegreesToRadians(xRotation));

    // Get the drag amount
    QPoint move = inputPosition() - previousInputPos();

    // Adjust axes
    switch (m_state) {
//! [0]
    case StateDraggingX:
        distance = (move.x() * xMulX - move.y() * xMulY) * m_speedModifier;
        m_axisXMinValue -= distance;
        m_axisXMaxValue -= distance;
        if (m_axisXMinValue < m_areaMinValue) {
            float dist = m_axisXMaxValue - m_axisXMinValue;
            m_axisXMinValue = m_areaMinValue;
            m_axisXMaxValue = m_axisXMinValue + dist;
        }
        if (m_axisXMaxValue > m_areaMaxValue) {
            float dist = m_axisXMaxValue - m_axisXMinValue;
            m_axisXMaxValue = m_areaMaxValue;
            m_axisXMinValue = m_axisXMaxValue - dist;
        }
        m_axisX->setRange(m_axisXMinValue, m_axisXMaxValue);
        break;
//! [0]
    case StateDraggingZ:
        distance = (move.x() * zMulX + move.y() * zMulY) * m_speedModifier;
        m_axisZMinValue += distance;
        m_axisZMaxValue += distance;
        if (m_axisZMinValue < m_areaMinValue) {
            float dist = m_axisZMaxValue - m_axisZMinValue;
            m_axisZMinValue = m_areaMinValue;
            m_axisZMaxValue = m_axisZMinValue + dist;
        }
        if (m_axisZMaxValue > m_areaMaxValue) {
            float dist = m_axisZMaxValue - m_axisZMinValue;
            m_axisZMaxValue = m_areaMaxValue;
            m_axisZMinValue = m_axisZMaxValue - dist;
        }
        m_axisZ->setRange(m_axisZMinValue, m_axisZMaxValue);
        break;
    default:
        break;
    }
}

void CustomInputHandler::checkConstraints()
{
//! [2]
    if (m_axisXMinValue < m_areaMinValue)
        m_axisXMinValue = m_areaMinValue;
    if (m_axisXMaxValue > m_areaMaxValue)
        m_axisXMaxValue = m_areaMaxValue;
    // Don't allow too much zoom in
    if ((m_axisXMaxValue - m_axisXMinValue) < m_axisXMinRange) {
        float adjust = (m_axisXMinRange - (m_axisXMaxValue - m_axisXMinValue)) / 2.0f;
        m_axisXMinValue -= adjust;
        m_axisXMaxValue += adjust;
    }
//! [2]

    if (m_axisZMinValue < m_areaMinValue)
        m_axisZMinValue = m_areaMinValue;
    if (m_axisZMaxValue > m_areaMaxValue)
        m_axisZMaxValue = m_areaMaxValue;
    // Don't allow too much zoom in
    if ((m_axisZMaxValue - m_axisZMinValue) < m_axisZMinRange) {
        float adjust = (m_axisZMinRange - (m_axisZMaxValue - m_axisZMinValue)) / 2.0f;
        m_axisZMinValue -= adjust;
        m_axisZMaxValue += adjust;
    }
}

/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#ifndef CUSTOMINPUTHANDLER_H
#define CUSTOMINPUTHANDLER_H

#include <QtDataVisualization/Q3DInputHandler>
#include <QtDataVisualization/QAbstract3DGraph>
#include <QtDataVisualization/QValue3DAxis>
#include "highlightseries.h"

class CustomInputHandler : public Q3DInputHandler
{
    Q_OBJECT

    enum InputState {
        StateNormal = 0,
        StateDraggingX,
        StateDraggingZ,
        StateDraggingY
    };

public:
    explicit CustomInputHandler(QAbstract3DGraph *graph, QObject *parent = 0);

    inline void setLimits(float min, float max, float minRange) {
        m_areaMinValue = min;
        m_areaMaxValue = max;
        m_axisXMinValue = m_areaMinValue;
        m_axisXMaxValue = m_areaMaxValue;
        m_axisZMinValue = m_areaMinValue;
        m_axisZMaxValue = m_areaMaxValue;
        m_axisXMinRange = minRange;
        m_axisZMinRange = minRange;
    }
    inline void setAxes(QValue3DAxis *axisX, QValue3DAxis *axisY, QValue3DAxis *axisZ) {
        m_axisX = axisX;
        m_axisY = axisY;
        m_axisZ = axisZ;
    }
    inline void setAspectRatio(float ratio) { m_aspectRatio = ratio; }
    inline void setHighlightSeries(HighlightSeries *series) { m_highlight = series; }
    inline void setDragSpeedModifier(float modifier) { m_speedModifier = modifier; }

    virtual void mousePressEvent(QMouseEvent *event, const QPoint &mousePos);
    virtual void mouseMoveEvent(QMouseEvent *event, const QPoint &mousePos);
    virtual void mouseReleaseEvent(QMouseEvent *event, const QPoint &mousePos);
    virtual void wheelEvent(QWheelEvent *event);

private:
    void handleElementSelected(QAbstract3DGraph::ElementType type);
    void handleAxisDragging();
    void checkConstraints();

private:
    HighlightSeries *m_highlight;
    bool m_mousePressed;
    InputState m_state;
    QValue3DAxis *m_axisX;
    QValue3DAxis *m_axisY;
    QValue3DAxis *m_axisZ;
    float m_speedModifier;
    float m_aspectRatio;
    float m_axisXMinValue;
    float m_axisXMaxValue;
    float m_axisXMinRange;
    float m_axisZMinValue;
    float m_axisZMaxValue;
    float m_axisZMinRange;
    float m_areaMinValue;
    float m_areaMaxValue;
};

#endif

/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#include "highlightseries.h"

//! [2]
const float darkRedPos = 1.0f;
const float redPos = 0.8f;
const float yellowPos = 0.6f;
const float greenPos = 0.4f;
const float darkGreenPos = 0.2f;
//! [2]

HighlightSeries::HighlightSeries()
    : m_width(100),
      m_height(100)
{
    setDrawMode(QSurface3DSeries::DrawSurface);
    setFlatShadingEnabled(true);
    setVisible(false);
}

HighlightSeries::~HighlightSeries()
{
}

//! [0]
void HighlightSeries::setTopographicSeries(TopographicSeries *series)
{
    m_topographicSeries = series;
    m_srcWidth = m_topographicSeries->dataProxy()->array()->at(0)->size();
    m_srcHeight = m_topographicSeries->dataProxy()->array()->size();

    QObject::connect(m_topographicSeries, &QSurface3DSeries::selectedPointChanged,
                     this, &HighlightSeries::handlePositionChange);
}
//! [0]

//! [1]
void HighlightSeries::handlePositionChange(const QPoint &position)
{
    m_position = position;

    if (position == invalidSelectionPosition()) {
        setVisible(false);

        return;
    }

    int halfWidth = m_width / 2;
    int halfHeight = m_height / 2;

    int startX = position.y() - halfWidth;
    if (startX < 0 )
        startX = 0;
    int endX = position.y() + halfWidth;
    if (endX > (m_srcWidth - 1))
        endX = m_srcWidth - 1;
    int startZ = position.x() - halfHeight;
    if (startZ < 0 )
        startZ = 0;
    int endZ = position.x() + halfHeight;
    if (endZ > (m_srcHeight - 1))
        endZ = m_srcHeight - 1;

    QSurfaceDataProxy *srcProxy = m_topographicSeries->dataProxy();
    const QSurfaceDataArray &srcArray = *srcProxy->array();

    QSurfaceDataArray *dataArray = new QSurfaceDataArray;
    dataArray->reserve(endZ - startZ);
    for (int i = startZ; i < endZ; i++) {
        QSurfaceDataRow *newRow = new QSurfaceDataRow(endX - startX);
        QSurfaceDataRow *srcRow = srcArray.at(i);
        for (int j = startX, p = 0; j < endX; j++, p++) {
            QVector3D pos = srcRow->at(j).position();
            (*newRow)[p].setPosition(QVector3D(pos.x(), pos.y() + 0.1f, pos.z()));
        }
        *dataArray << newRow;
    }

    dataProxy()->resetArray(dataArray);
    setVisible(true);
}
//! [1]

//! [3]
void HighlightSeries::handleGradientChange(float value)
{
    float ratio = m_minHeight / value;

    QLinearGradient gr;
    gr.setColorAt(0.0f, Qt::black);
    gr.setColorAt(darkGreenPos * ratio, Qt::darkGreen);
    gr.setColorAt(greenPos * ratio, Qt::green);
    gr.setColorAt(yellowPos * ratio, Qt::yellow);
    gr.setColorAt(redPos * ratio, Qt::red);
    gr.setColorAt(darkRedPos * ratio, Qt::darkRed);

    setBaseGradient(gr);
    setColorStyle(Q3DTheme::ColorStyleRangeGradient);
}
//! [3]

/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#ifndef HIGHLIGHTSERIES_H
#define HIGHLIGHTSERIES_H

#include <QtDataVisualization/QSurface3DSeries>

#include "topographicseries.h"

class HighlightSeries : public QSurface3DSeries
{
    Q_OBJECT
public:
    explicit HighlightSeries();
    ~HighlightSeries();

    void setTopographicSeries(TopographicSeries *series);
    inline void setMinHeight(float height) { m_minHeight = height; }

public Q_SLOTS:
    void handlePositionChange(const QPoint &position);
    void handleGradientChange(float value);

private:
    int m_width;
    int m_height;
    int m_srcWidth;
    int m_srcHeight;
    QPoint m_position;
    TopographicSeries *m_topographicSeries;
    float m_minHeight;
};

#endif // HIGHLIGHTSERIES_H

License information regarding the data obtained from National Land Survey of
Finland http://www.maanmittauslaitos.fi/en
- topographic model from Elevation model 2 m (U4421B, U4421D, U4422A and
  U4422C) 08/2014
- map image extracted from Topographic map raster 1:50 000 (U442) 08/2014

National Land Survey open data licence - version 1.0 - 1 May 2012

1. General information

The National Land Survey of Finland (hereinafter the Licensor), as the holder
of the immaterial rights to the data, has granted on the terms mentioned below
the right to use a copy (hereinafter data or dataset(s)) of the data (or a part
of it).

The Licensee is a natural or legal person who makes use of the data covered by
this licence. The Licensee accepts the terms of this licence by receiving the
dataset(s) covered by the licence.

This Licence agreement does not create a co-operation or business relationship
between the Licensee and the Licensor.

2. Terms of the licence

2.1. Right of use

This licence grants a worldwide, free of charge and irrevocable parallel right
of use to open data. According to the terms of the licence, data received by
the Licensee can be freely:
 - copied, distributed and published,
 - modified and utilised commercially and non-commercially,
 - inserted into other products and
 - used as a part of a software application or service.

2.2. Duties and responsibilities of the Licensee

Through reasonable means suitable to the distribution medium or method which is
used in conjunction with a product containing data or a service utilising data
covered by this licence or while distributing data, the Licensee shall:
 - mention the name of the Licensor, the name of the dataset(s) and the time
   when the National Land Survey has delivered the dataset(s) (e.g.: contains
   data from the National Land Survey of Finland Topographic Database 06/2012)
 - provide a copy of this licence or a link to it, as well as
 - require third parties to provide the same information when granting rights
   to copies of dataset(s) or products and services containing such data and
 - remove the name of the Licensor from the product or service, if required to
   do so by the Licensor.

The terms of this licence do not allow the Licensee to state in conjunction
with the use of dataset(s) that the Licensor supports or recommends such use.

2.3. Duties and responsibilities of the Licensor

The Licensor shall ensure that
 - the Licensor has the right to grant rights to the dataset(s) in accordance
   with this licence.

The data has been licensed "as is" and the Licensor
 - shall not be held responsible for any errors or omissions in the data,
   disclaims any warranty for the validity or up to date status of the data and
   shall be free from liability for direct or consequential damages arising
   from the use of data provided by the Licensor,
 - and is not obligated to ensure the continuous availability of the data, nor
   to announce in advance the interruption or cessation of availability, and
   the Licensor shall be free from liability for direct or consequential
   damages arising from any such interruption or cessation.

3. Jurisdiction

Finnish law shall apply to this licence.

4. Changes to this licence

The Licensor may at any time change the terms of the licence or apply a
different licence to the data. The terms of this licence shall, however, still
apply to such data that has been received prior to the change of the terms of
the licence or the licence itself.

/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#include "surfacegraph.h"

#include <QtWidgets/QApplication>
#include <QtWidgets/QWidget>
#include <QtWidgets/QHBoxLayout>
#include <QtWidgets/QVBoxLayout>
#include <QtWidgets/QGroupBox>
#include <QtWidgets/QCheckBox>
#include <QtWidgets/QLabel>
#include <QtWidgets/QMessageBox>
#include <QtGui/QScreen>
#include <QtGui/QPainter>

int main(int argc, char **argv)
{
    QApplication app(argc, argv);
    Q3DSurface *graph = new Q3DSurface();
    QWidget *container = QWidget::createWindowContainer(graph);

    if (!graph->hasContext()) {
        QMessageBox msgBox;
        msgBox.setText("Couldn't initialize the OpenGL context.");
        msgBox.exec();
        return -1;
    }

    QSize screenSize = graph->screen()->size();
    container->setMinimumSize(QSize(screenSize.width() / 2, screenSize.height() / 1.6));
    container->setMaximumSize(screenSize);
    container->setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Expanding);
    container->setFocusPolicy(Qt::StrongFocus);

    QWidget *widget = new QWidget;
    QHBoxLayout *hLayout = new QHBoxLayout(widget);
    QVBoxLayout *vLayout = new QVBoxLayout();
    hLayout->addWidget(container, 1);
    hLayout->addLayout(vLayout);
    vLayout->setAlignment(Qt::AlignTop);

    widget->setWindowTitle(QStringLiteral("Textured surface example"));

    QCheckBox *enableTexture = new QCheckBox(widget);
    enableTexture->setText(QStringLiteral("Surface texture"));

    int height = 400;
    int width = 100;
    int border = 10;
    QLinearGradient gr(0, 0, 1, height - 2 * border);
    gr.setColorAt(1.0f, Qt::black);
    gr.setColorAt(0.8f, Qt::darkGreen);
    gr.setColorAt(0.6f, Qt::green);
    gr.setColorAt(0.4f, Qt::yellow);
    gr.setColorAt(0.2f, Qt::red);
    gr.setColorAt(0.0f, Qt::darkRed);

    QPixmap pm(width, height);
    pm.fill(Qt::transparent);
    QPainter pmp(&pm);
    pmp.setBrush(QBrush(gr));
    pmp.setPen(Qt::NoPen);
    pmp.drawRect(border, border, 35, height - 2 * border);
    pmp.setPen(Qt::black);
    int step = (height - 2 * border) / 5;
    for (int i = 0; i < 6; i++) {
        int yPos = i * step + border;
        pmp.drawLine(border, yPos, 55, yPos);
        pmp.drawText(60, yPos + 2, QString("%1 m").arg(550 - (i * 110)));
    }

    QLabel *label = new QLabel(widget);
    label->setPixmap(pm);

    QGroupBox *heightMapGroupBox = new QGroupBox(QStringLiteral("Highlight color map"));
    QVBoxLayout *colorMapVBox = new QVBoxLayout;
    colorMapVBox->addWidget(label);
    heightMapGroupBox->setLayout(colorMapVBox);

    vLayout->addWidget(enableTexture);
    vLayout->addWidget(heightMapGroupBox);

    widget->show();

    SurfaceGraph *modifier = new SurfaceGraph(graph);

    QObject::connect(enableTexture, &QCheckBox::stateChanged,
                     modifier, &SurfaceGraph::toggleSurfaceTexture);

    enableTexture->setChecked(true);

    return app.exec();
}

#############################################################################
# Makefile for building: texturesurface
# Generated by qmake (3.1) (Qt 5.14.2)
# Project:  texturesurface.pro
# Template: app
# Command: D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -o Makefile texturesurface.pro -spec win32-msvc "CONFIG+=debug" "CONFIG+=qml_debug"
#############################################################################

MAKEFILE      = Makefile

EQ            = =

first: debug
install: debug-install
uninstall: debug-uninstall
QMAKE         = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe
DEL_FILE      = del
CHK_DIR_EXISTS= if not exist
MKDIR         = mkdir
COPY          = copy /y
COPY_FILE     = copy /y
COPY_DIR      = xcopy /s /q /y /i
INSTALL_FILE  = copy /y
INSTALL_PROGRAM = copy /y
INSTALL_DIR   = xcopy /s /q /y /i
QINSTALL      = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -install qinstall
QINSTALL_PROGRAM = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -install qinstall -exe
DEL_FILE      = del
SYMLINK       = $(QMAKE) -install ln -f -s
DEL_DIR       = rmdir
MOVE          = move
IDC           = idc
IDL           = midl
ZIP           = zip -r -9
DEF_FILE      = 
RES_FILE      = 
SED           = $(QMAKE) -install sed
MOVE          = move
SUBTARGETS    =  \
		debug \
		release


debug: $(MAKEFILE) FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug
debug-make_first: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug 
debug-all: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug all
debug-clean: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug clean
debug-distclean: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug distclean
debug-install: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug install
debug-uninstall: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug uninstall
release: $(MAKEFILE) FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release
release-make_first: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release 
release-all: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release all
release-clean: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release clean
release-distclean: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release distclean
release-install: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release install
release-uninstall: FORCE
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release uninstall

Makefile: texturesurface.pro ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\win32-msvc\qmake.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\spec_pre.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\angle.conf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\windows-desktop.conf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\windows_vulkan_sdk.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\windows-vulkan.conf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\msvc-desktop.conf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\qconfig.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3danimation.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3danimation_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dcore.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dcore_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dextras.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dextras_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dinput.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dinput_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dlogic.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dlogic_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquick.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquick_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickanimation.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickanimation_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickextras.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickextras_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickinput.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickinput_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickrender.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickrender_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickscene2d.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickscene2d_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3drender.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3drender_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_accessibility_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axbase.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axbase_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axcontainer.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axcontainer_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axserver.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axserver_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bluetooth.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bluetooth_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bootstrap_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_charts.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_charts_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_concurrent.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_concurrent_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_core.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_core_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_datavisualization.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_datavisualization_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_dbus.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_dbus_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designer.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designer_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designercomponents_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_devicediscovery_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_edid_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_egl_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_eventdispatcher_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_fb_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_fontdatabase_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gamepad.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gamepad_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gui.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gui_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_help.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_help_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_location.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_location_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimedia.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimedia_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimediawidgets.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimediawidgets_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_network.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_network_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_networkauth.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_networkauth_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_nfc.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_nfc_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_opengl.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_opengl_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_openglextensions.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_openglextensions_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_packetprotocol_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_platformcompositor_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioning.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioning_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioningquick.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioningquick_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_printsupport.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_printsupport_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qml.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qml_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmldebug_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmldevtools_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlmodels.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlmodels_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmltest.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmltest_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlworkerscript.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlworkerscript_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qtmultimediaquicktools_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quick.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quick_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickcontrols2.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickcontrols2_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickparticles_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickshapes_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quicktemplates2.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quicktemplates2_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickwidgets.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickwidgets_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_remoteobjects.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_remoteobjects_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_repparser.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_repparser_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_scxml.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_scxml_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sensors.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sensors_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialbus.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialbus_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialport.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialport_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sql.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sql_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_svg.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_svg_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_testlib.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_testlib_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_texttospeech.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_texttospeech_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_theme_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uiplugin.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uitools.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uitools_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_virtualkeyboard.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_virtualkeyboard_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_vulkan_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webchannel.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webchannel_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_websockets.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_websockets_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webview.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webview_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_widgets.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_widgets_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_windowsuiautomation_support_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_winextras.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_winextras_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xml.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xml_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xmlpatterns.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xmlpatterns_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_zlib_private.pri \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt_functions.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt_config.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\win32-msvc\qmake.conf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\spec_post.prf \
		.qmake.stash \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exclusive_builds.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\msvc-version.conf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\toolchain.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\default_pre.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\default_pre.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resolve_config.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exclusive_builds_post.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\default_post.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qml_debug.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\precompile_header.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\warn_on.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resources_functions.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resources.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\moc.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\opengl.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\uic.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qmake_use.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\file_copies.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\windows.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\testcase_targets.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exceptions.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\yacc.prf \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\lex.prf \
		texturesurface.pro \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgets.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualization.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Gui.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Core.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmain.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\build_pass.prf \
		texturedsurface.qrc \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgetsd.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualizationd.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Guid.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Cored.prl \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmaind.prl
	$(QMAKE) -o Makefile texturesurface.pro -spec win32-msvc "CONFIG+=debug" "CONFIG+=qml_debug"
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\spec_pre.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\angle.conf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\windows-desktop.conf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\windows_vulkan_sdk.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\windows-vulkan.conf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\msvc-desktop.conf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\qconfig.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3danimation.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3danimation_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dcore.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dcore_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dextras.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dextras_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dinput.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dinput_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dlogic.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dlogic_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquick.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquick_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickanimation.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickanimation_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickextras.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickextras_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickinput.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickinput_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickrender.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickrender_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickscene2d.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickscene2d_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3drender.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3drender_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_accessibility_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axbase.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axbase_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axcontainer.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axcontainer_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axserver.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axserver_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bluetooth.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bluetooth_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bootstrap_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_charts.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_charts_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_concurrent.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_concurrent_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_core.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_core_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_datavisualization.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_datavisualization_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_dbus.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_dbus_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designer.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designer_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designercomponents_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_devicediscovery_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_edid_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_egl_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_eventdispatcher_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_fb_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_fontdatabase_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gamepad.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gamepad_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gui.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gui_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_help.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_help_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_location.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_location_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimedia.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimedia_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimediawidgets.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimediawidgets_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_network.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_network_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_networkauth.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_networkauth_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_nfc.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_nfc_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_opengl.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_opengl_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_openglextensions.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_openglextensions_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_packetprotocol_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_platformcompositor_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioning.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioning_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioningquick.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioningquick_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_printsupport.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_printsupport_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qml.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qml_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmldebug_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmldevtools_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlmodels.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlmodels_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmltest.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmltest_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlworkerscript.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlworkerscript_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qtmultimediaquicktools_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quick.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quick_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickcontrols2.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickcontrols2_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickparticles_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickshapes_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quicktemplates2.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quicktemplates2_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickwidgets.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickwidgets_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_remoteobjects.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_remoteobjects_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_repparser.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_repparser_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_scxml.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_scxml_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sensors.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sensors_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialbus.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialbus_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialport.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialport_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sql.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sql_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_svg.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_svg_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_testlib.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_testlib_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_texttospeech.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_texttospeech_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_theme_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uiplugin.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uitools.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uitools_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_virtualkeyboard.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_virtualkeyboard_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_vulkan_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webchannel.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webchannel_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_websockets.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_websockets_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webview.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webview_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_widgets.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_widgets_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_windowsuiautomation_support_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_winextras.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_winextras_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xml.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xml_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xmlpatterns.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xmlpatterns_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_zlib_private.pri:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt_functions.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt_config.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\win32-msvc\qmake.conf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\spec_post.prf:
.qmake.stash:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exclusive_builds.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\msvc-version.conf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\toolchain.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\default_pre.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\default_pre.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resolve_config.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exclusive_builds_post.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\default_post.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qml_debug.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\precompile_header.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\warn_on.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resources_functions.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resources.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\moc.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\opengl.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\uic.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qmake_use.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\file_copies.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\windows.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\testcase_targets.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exceptions.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\yacc.prf:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\lex.prf:
texturesurface.pro:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgets.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualization.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Gui.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Core.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmain.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\build_pass.prf:
texturedsurface.qrc:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgetsd.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualizationd.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Guid.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Cored.prl:
..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmaind.prl:
qmake: FORCE
	@$(QMAKE) -o Makefile texturesurface.pro -spec win32-msvc "CONFIG+=debug" "CONFIG+=qml_debug"

qmake_all: FORCE

make_first: debug-make_first release-make_first  FORCE
all: debug-all release-all  FORCE
clean: debug-clean release-clean  FORCE
	-$(DEL_FILE) texturesurface.vc.pdb
	-$(DEL_FILE) texturesurface.ilk
	-$(DEL_FILE) texturesurface.idb
distclean: debug-distclean release-distclean  FORCE
	-$(DEL_FILE) Makefile
	-$(DEL_FILE) .qmake.stash texturesurface.pdb

debug-mocclean:
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug mocclean
release-mocclean:
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release mocclean
mocclean: debug-mocclean release-mocclean

debug-mocables:
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Debug mocables
release-mocables:
	@set MAKEFLAGS=$(MAKEFLAGS)
	$(MAKE) -f $(MAKEFILE).Release mocables
mocables: debug-mocables release-mocables

check: first

benchmark: first
FORCE:

$(MAKEFILE).Debug: Makefile
$(MAKEFILE).Release: Makefile

#############################################################################
# Makefile for building: texturesurface
# Generated by qmake (3.1) (Qt 5.14.2)
# Project:  texturesurface.pro
# Template: app
#############################################################################

MAKEFILE      = Makefile.Debug

EQ            = =

####### Compiler, tools and options

CC            = cl
CXX           = cl
DEFINES       = -DUNICODE -D_UNICODE -DWIN32 -D_ENABLE_EXTENDED_ALIGNED_STORAGE -DWIN64 -DQT_QML_DEBUG -DQT_WIDGETS_LIB -DQT_DATAVISUALIZATION_LIB -DQT_GUI_LIB -DQT_CORE_LIB
CFLAGS        = -nologo -Zc:wchar_t -FS -Zc:strictStrings -Zi -MDd -W3 -w44456 -w44457 -w44458 /Fddebug\texturesurface.vc.pdb $(DEFINES)
CXXFLAGS      = -nologo -Zc:wchar_t -FS -Zc:rvalueCast -Zc:inline -Zc:strictStrings -Zc:throwingNew -Zc:referenceBinding -Zc:__cplusplus -Zi -MDd -W3 -w34100 -w34189 -w44996 -w44456 -w44457 -w44458 -wd4577 -wd4467 -EHsc /Fddebug\texturesurface.vc.pdb $(DEFINES)
INCPATH       = -I. -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore -Idebug -I/include -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\win32-msvc 
LINKER        = link
LFLAGS        = /NOLOGO /DYNAMICBASE /NXCOMPAT /DEBUG /SUBSYSTEM:WINDOWS "/MANIFESTDEPENDENCY:type='win32' name='Microsoft.Windows.Common-Controls' version='6.0.0.0' publicKeyToken='6595b64144ccf1df' language='*' processorArchitecture='*'"
LIBS          = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgetsd.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualizationd.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Guid.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Cored.lib  D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmaind.lib /LIBPATH:C:\openssl\lib /LIBPATH:C:\Utils\my_sql\mysql-5.7.25-winx64\lib /LIBPATH:C:\Utils\postgresql\pgsql\lib shell32.lib 
QMAKE         = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe
DEL_FILE      = del
CHK_DIR_EXISTS= if not exist
MKDIR         = mkdir
COPY          = copy /y
COPY_FILE     = copy /y
COPY_DIR      = xcopy /s /q /y /i
INSTALL_FILE  = copy /y
INSTALL_PROGRAM = copy /y
INSTALL_DIR   = xcopy /s /q /y /i
QINSTALL      = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -install qinstall
QINSTALL_PROGRAM = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -install qinstall -exe
DEL_FILE      = del
SYMLINK       = $(QMAKE) -install ln -f -s
DEL_DIR       = rmdir
MOVE          = move
IDC           = idc
IDL           = midl
ZIP           = zip -r -9
DEF_FILE      = 
RES_FILE      = 
SED           = $(QMAKE) -install sed
MOVE          = move

####### Output directory

OBJECTS_DIR   = debug

####### Files

SOURCES       = main.cpp \
		surfacegraph.cpp \
		topographicseries.cpp \
		highlightseries.cpp \
		custominputhandler.cpp debug\qrc_texturedsurface.cpp \
		debug\moc_surfacegraph.cpp \
		debug\moc_topographicseries.cpp \
		debug\moc_highlightseries.cpp \
		debug\moc_custominputhandler.cpp
OBJECTS       = debug\main.obj \
		debug\surfacegraph.obj \
		debug\topographicseries.obj \
		debug\highlightseries.obj \
		debug\custominputhandler.obj \
		debug\qrc_texturedsurface.obj \
		debug\moc_surfacegraph.obj \
		debug\moc_topographicseries.obj \
		debug\moc_highlightseries.obj \
		debug\moc_custominputhandler.obj

DIST          =  surfacegraph.h \
		topographicseries.h \
		highlightseries.h \
		custominputhandler.h main.cpp \
		surfacegraph.cpp \
		topographicseries.cpp \
		highlightseries.cpp \
		custominputhandler.cpp
QMAKE_TARGET  = texturesurface
DESTDIR        = debug\ #avoid trailing-slash linebreak
TARGET         = texturesurface.exe
DESTDIR_TARGET = debug\texturesurface.exe

####### Implicit rules

.SUFFIXES: .c .cpp .cc .cxx

{debug}.cpp{debug\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Fodebug\ @<<
	$<
<<

{debug}.cc{debug\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Fodebug\ @<<
	$<
<<

{debug}.cxx{debug\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Fodebug\ @<<
	$<
<<

{debug}.c{debug\}.obj::
	$(CC) -c $(CFLAGS) $(INCPATH) -Fodebug\ @<<
	$<
<<

{.}.cpp{debug\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Fodebug\ @<<
	$<
<<

{.}.cc{debug\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Fodebug\ @<<
	$<
<<

{.}.cxx{debug\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Fodebug\ @<<
	$<
<<

{.}.c{debug\}.obj::
	$(CC) -c $(CFLAGS) $(INCPATH) -Fodebug\ @<<
	$<
<<

####### Build rules

first: all
all: Makefile.Debug  debug\texturesurface.exe

debug\texturesurface.exe: D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgetsd.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualizationd.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Guid.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Cored.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmaind.lib $(OBJECTS) 
	$(LINKER) $(LFLAGS) /MANIFEST:embed /OUT:$(DESTDIR_TARGET) @<<
debug\main.obj debug\surfacegraph.obj debug\topographicseries.obj debug\highlightseries.obj debug\custominputhandler.obj debug\qrc_texturedsurface.obj debug\moc_surfacegraph.obj debug\moc_topographicseries.obj debug\moc_highlightseries.obj debug\moc_custominputhandler.obj
$(LIBS)
<<

qmake: FORCE
	@$(QMAKE) -o Makefile.Debug texturesurface.pro -spec win32-msvc "CONFIG+=debug" "CONFIG+=qml_debug"

qmake_all: FORCE

dist:
	$(ZIP) texturesurface.zip $(SOURCES) $(DIST) texturesurface.pro ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\spec_pre.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\angle.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\windows-desktop.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\windows_vulkan_sdk.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\windows-vulkan.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\msvc-desktop.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\qconfig.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3danimation.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3danimation_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dcore.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dcore_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dextras.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dextras_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dinput.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dinput_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dlogic.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dlogic_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquick.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquick_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickanimation.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickanimation_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickextras.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickextras_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickinput.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickinput_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickrender.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickrender_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickscene2d.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickscene2d_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3drender.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3drender_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_accessibility_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axbase.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axbase_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axcontainer.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axcontainer_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axserver.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axserver_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bluetooth.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bluetooth_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bootstrap_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_charts.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_charts_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_concurrent.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_concurrent_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_core.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_core_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_datavisualization.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_datavisualization_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_dbus.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_dbus_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designer.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designer_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designercomponents_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_devicediscovery_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_edid_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_egl_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_eventdispatcher_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_fb_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_fontdatabase_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gamepad.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gamepad_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gui.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gui_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_help.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_help_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_location.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_location_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimedia.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimedia_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimediawidgets.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimediawidgets_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_network.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_network_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_networkauth.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_networkauth_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_nfc.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_nfc_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_opengl.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_opengl_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_openglextensions.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_openglextensions_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_packetprotocol_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_platformcompositor_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioning.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioning_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioningquick.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioningquick_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_printsupport.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_printsupport_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qml.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qml_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmldebug_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmldevtools_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlmodels.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlmodels_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmltest.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmltest_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlworkerscript.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlworkerscript_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qtmultimediaquicktools_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quick.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quick_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickcontrols2.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickcontrols2_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickparticles_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickshapes_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quicktemplates2.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quicktemplates2_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickwidgets.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickwidgets_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_remoteobjects.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_remoteobjects_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_repparser.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_repparser_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_scxml.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_scxml_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sensors.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sensors_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialbus.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialbus_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialport.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialport_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sql.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sql_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_svg.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_svg_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_testlib.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_testlib_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_texttospeech.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_texttospeech_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_theme_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uiplugin.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uitools.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uitools_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_virtualkeyboard.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_virtualkeyboard_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_vulkan_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webchannel.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webchannel_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_websockets.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_websockets_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webview.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webview_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_widgets.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_widgets_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_windowsuiautomation_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_winextras.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_winextras_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xml.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xml_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xmlpatterns.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xmlpatterns_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_zlib_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt_functions.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt_config.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\win32-msvc\qmake.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\spec_post.prf .qmake.stash ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exclusive_builds.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\msvc-version.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\toolchain.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\default_pre.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\default_pre.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resolve_config.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exclusive_builds_post.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\default_post.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\build_pass.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qml_debug.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\precompile_header.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\warn_on.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resources_functions.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resources.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\moc.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\opengl.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\uic.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qmake_use.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\file_copies.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\windows.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\testcase_targets.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exceptions.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\yacc.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\lex.prf texturesurface.pro texturedsurface.qrc ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgetsd.prl ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualizationd.prl ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Guid.prl ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Cored.prl ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmaind.prl   texturedsurface.qrc ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\data\dummy.cpp surfacegraph.h topographicseries.h highlightseries.h custominputhandler.h  main.cpp surfacegraph.cpp topographicseries.cpp highlightseries.cpp custominputhandler.cpp     

clean: compiler_clean 
	-$(DEL_FILE) debug\main.obj debug\surfacegraph.obj debug\topographicseries.obj debug\highlightseries.obj debug\custominputhandler.obj debug\qrc_texturedsurface.obj debug\moc_surfacegraph.obj debug\moc_topographicseries.obj debug\moc_highlightseries.obj debug\moc_custominputhandler.obj
	-$(DEL_FILE) debug\texturesurface.vc.pdb debug\texturesurface.ilk debug\texturesurface.idb

distclean: clean 
	-$(DEL_FILE) .qmake.stash debug\texturesurface.pdb
	-$(DEL_FILE) $(DESTDIR_TARGET)
	-$(DEL_FILE) Makefile.Debug

mocclean: compiler_moc_header_clean compiler_moc_objc_header_clean compiler_moc_source_clean

mocables: compiler_moc_header_make_all compiler_moc_objc_header_make_all compiler_moc_source_make_all

check: first

benchmark: first

compiler_no_pch_compiler_make_all:
compiler_no_pch_compiler_clean:
compiler_rcc_make_all: debug\qrc_texturedsurface.cpp
compiler_rcc_clean:
	-$(DEL_FILE) debug\qrc_texturedsurface.cpp
debug\qrc_texturedsurface.cpp: texturedsurface.qrc \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\rcc.exe \
		topography.png \
		maptexture.jpg
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\rcc.exe -name texturedsurface texturedsurface.qrc -o debug\qrc_texturedsurface.cpp

compiler_moc_predefs_make_all: debug\moc_predefs.h
compiler_moc_predefs_clean:
	-$(DEL_FILE) debug\moc_predefs.h
debug\moc_predefs.h: ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\data\dummy.cpp
	cl -BxD:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -nologo -Zc:wchar_t -FS -Zc:rvalueCast -Zc:inline -Zc:strictStrings -Zc:throwingNew -Zc:referenceBinding -Zc:__cplusplus -Zi -MDd -W3 -w34100 -w34189 -w44996 -w44456 -w44457 -w44458 -wd4577 -wd4467 -E ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\data\dummy.cpp 2>NUL >debug\moc_predefs.h

compiler_moc_header_make_all: debug\moc_surfacegraph.cpp debug\moc_topographicseries.cpp debug\moc_highlightseries.cpp debug\moc_custominputhandler.cpp
compiler_moc_header_clean:
	-$(DEL_FILE) debug\moc_surfacegraph.cpp debug\moc_topographicseries.cpp debug\moc_highlightseries.cpp debug\moc_custominputhandler.cpp
debug\moc_surfacegraph.cpp: surfacegraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DSurface \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QSlider \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgetsglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgets-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qabstractslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qwidget.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpalette.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontmetrics.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qsizepolicy.h \
		topographicseries.h \
		highlightseries.h \
		custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		debug\moc_predefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe $(DEFINES) --compiler-flavor=msvc --include D:/PCLcode/texturesurface/texturesurface/debug/moc_predefs.h -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/mkspecs/win32-msvc -ID:/PCLcode/texturesurface/texturesurface -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtWidgets -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtDataVisualization -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtGui -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtANGLE -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtCore -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\ATLMFC\include" -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\include" -I"C:\Program Files (x86)\Windows Kits\NETFXSDK\4.6.1\include\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\ucrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\shared" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\winrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\cppwinrt" surfacegraph.h -o debug\moc_surfacegraph.cpp

debug\moc_topographicseries.cpp: topographicseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		debug\moc_predefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe $(DEFINES) --compiler-flavor=msvc --include D:/PCLcode/texturesurface/texturesurface/debug/moc_predefs.h -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/mkspecs/win32-msvc -ID:/PCLcode/texturesurface/texturesurface -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtWidgets -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtDataVisualization -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtGui -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtANGLE -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtCore -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\ATLMFC\include" -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\include" -I"C:\Program Files (x86)\Windows Kits\NETFXSDK\4.6.1\include\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\ucrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\shared" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\winrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\cppwinrt" topographicseries.h -o debug\moc_topographicseries.cpp

debug\moc_highlightseries.cpp: highlightseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		topographicseries.h \
		debug\moc_predefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe $(DEFINES) --compiler-flavor=msvc --include D:/PCLcode/texturesurface/texturesurface/debug/moc_predefs.h -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/mkspecs/win32-msvc -ID:/PCLcode/texturesurface/texturesurface -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtWidgets -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtDataVisualization -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtGui -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtANGLE -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtCore -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\ATLMFC\include" -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\include" -I"C:\Program Files (x86)\Windows Kits\NETFXSDK\4.6.1\include\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\ucrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\shared" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\winrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\cppwinrt" highlightseries.h -o debug\moc_highlightseries.cpp

debug\moc_custominputhandler.cpp: custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		highlightseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		topographicseries.h \
		debug\moc_predefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe $(DEFINES) --compiler-flavor=msvc --include D:/PCLcode/texturesurface/texturesurface/debug/moc_predefs.h -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/mkspecs/win32-msvc -ID:/PCLcode/texturesurface/texturesurface -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtWidgets -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtDataVisualization -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtGui -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtANGLE -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtCore -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\ATLMFC\include" -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\include" -I"C:\Program Files (x86)\Windows Kits\NETFXSDK\4.6.1\include\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\ucrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\shared" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\winrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\cppwinrt" custominputhandler.h -o debug\moc_custominputhandler.cpp

compiler_moc_objc_header_make_all:
compiler_moc_objc_header_clean:
compiler_moc_source_make_all:
compiler_moc_source_clean:
compiler_uic_make_all:
compiler_uic_clean:
compiler_yacc_decl_make_all:
compiler_yacc_decl_clean:
compiler_yacc_impl_make_all:
compiler_yacc_impl_clean:
compiler_lex_make_all:
compiler_lex_clean:
compiler_clean: compiler_rcc_clean compiler_moc_predefs_clean compiler_moc_header_clean 



####### Compile

debug\main.obj: main.cpp surfacegraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DSurface \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QSlider \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgetsglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgets-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qabstractslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qwidget.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpalette.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontmetrics.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qsizepolicy.h \
		topographicseries.h \
		highlightseries.h \
		custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QApplication \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qapplication.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreapplication.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qeventloop.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qdesktopwidget.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qguiapplication.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qinputmethod.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QWidget \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QHBoxLayout \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qboxlayout.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qlayout.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qlayoutitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qgridlayout.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QVBoxLayout \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QGroupBox \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qgroupbox.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qframe.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QCheckBox \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qcheckbox.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qabstractbutton.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QLabel \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qlabel.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QMessageBox \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qmessagebox.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qdialog.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QScreen \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qscreen.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QSize \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QSizeF \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTransform \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QPainter \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtextoption.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpen.h

debug\surfacegraph.obj: surfacegraph.cpp surfacegraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DSurface \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QSlider \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgetsglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgets-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qabstractslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qwidget.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpalette.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontmetrics.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qsizepolicy.h \
		topographicseries.h \
		highlightseries.h \
		custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DTheme

debug\topographicseries.obj: topographicseries.cpp topographicseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D

debug\highlightseries.obj: highlightseries.cpp highlightseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		topographicseries.h

debug\custominputhandler.obj: custominputhandler.cpp custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		highlightseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		topographicseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DCamera \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmath.h

debug\qrc_texturedsurface.obj: debug\qrc_texturedsurface.cpp 

debug\moc_surfacegraph.obj: debug\moc_surfacegraph.cpp 

debug\moc_topographicseries.obj: debug\moc_topographicseries.cpp 

debug\moc_highlightseries.obj: debug\moc_highlightseries.cpp 

debug\moc_custominputhandler.obj: debug\moc_custominputhandler.cpp 

####### Install

install:  FORCE

uninstall:  FORCE

FORCE:


#############################################################################
# Makefile for building: texturesurface
# Generated by qmake (3.1) (Qt 5.14.2)
# Project:  texturesurface.pro
# Template: app
#############################################################################

MAKEFILE      = Makefile.Release

EQ            = =

####### Compiler, tools and options

CC            = cl
CXX           = cl
DEFINES       = -DUNICODE -D_UNICODE -DWIN32 -D_ENABLE_EXTENDED_ALIGNED_STORAGE -DWIN64 -DNDEBUG -DQT_QML_DEBUG -DQT_NO_DEBUG -DQT_WIDGETS_LIB -DQT_DATAVISUALIZATION_LIB -DQT_GUI_LIB -DQT_CORE_LIB
CFLAGS        = -nologo -Zc:wchar_t -FS -Zc:strictStrings -O2 -MD -W3 -w44456 -w44457 -w44458 $(DEFINES)
CXXFLAGS      = -nologo -Zc:wchar_t -FS -Zc:rvalueCast -Zc:inline -Zc:strictStrings -Zc:throwingNew -Zc:referenceBinding -Zc:__cplusplus -O2 -MD -W3 -w34100 -w34189 -w44996 -w44456 -w44457 -w44458 -wd4577 -wd4467 -EHsc $(DEFINES)
INCPATH       = -I. -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore -Irelease -I/include -I..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\win32-msvc 
LINKER        = link
LFLAGS        = /NOLOGO /DYNAMICBASE /NXCOMPAT /OPT:REF /INCREMENTAL:NO /SUBSYSTEM:WINDOWS "/MANIFESTDEPENDENCY:type='win32' name='Microsoft.Windows.Common-Controls' version='6.0.0.0' publicKeyToken='6595b64144ccf1df' language='*' processorArchitecture='*'"
LIBS          = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgets.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualization.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Gui.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Core.lib  D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmain.lib /LIBPATH:C:\openssl\lib /LIBPATH:C:\Utils\my_sql\mysql-5.7.25-winx64\lib /LIBPATH:C:\Utils\postgresql\pgsql\lib shell32.lib 
QMAKE         = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe
DEL_FILE      = del
CHK_DIR_EXISTS= if not exist
MKDIR         = mkdir
COPY          = copy /y
COPY_FILE     = copy /y
COPY_DIR      = xcopy /s /q /y /i
INSTALL_FILE  = copy /y
INSTALL_PROGRAM = copy /y
INSTALL_DIR   = xcopy /s /q /y /i
QINSTALL      = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -install qinstall
QINSTALL_PROGRAM = D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -install qinstall -exe
DEL_FILE      = del
SYMLINK       = $(QMAKE) -install ln -f -s
DEL_DIR       = rmdir
MOVE          = move
IDC           = idc
IDL           = midl
ZIP           = zip -r -9
DEF_FILE      = 
RES_FILE      = 
SED           = $(QMAKE) -install sed
MOVE          = move

####### Output directory

OBJECTS_DIR   = release

####### Files

SOURCES       = main.cpp \
		surfacegraph.cpp \
		topographicseries.cpp \
		highlightseries.cpp \
		custominputhandler.cpp release\qrc_texturedsurface.cpp \
		release\moc_surfacegraph.cpp \
		release\moc_topographicseries.cpp \
		release\moc_highlightseries.cpp \
		release\moc_custominputhandler.cpp
OBJECTS       = release\main.obj \
		release\surfacegraph.obj \
		release\topographicseries.obj \
		release\highlightseries.obj \
		release\custominputhandler.obj \
		release\qrc_texturedsurface.obj \
		release\moc_surfacegraph.obj \
		release\moc_topographicseries.obj \
		release\moc_highlightseries.obj \
		release\moc_custominputhandler.obj

DIST          =  surfacegraph.h \
		topographicseries.h \
		highlightseries.h \
		custominputhandler.h main.cpp \
		surfacegraph.cpp \
		topographicseries.cpp \
		highlightseries.cpp \
		custominputhandler.cpp
QMAKE_TARGET  = texturesurface
DESTDIR        = release\ #avoid trailing-slash linebreak
TARGET         = texturesurface.exe
DESTDIR_TARGET = release\texturesurface.exe

####### Implicit rules

.SUFFIXES: .c .cpp .cc .cxx

{release}.cpp{release\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Forelease\ @<<
	$<
<<

{release}.cc{release\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Forelease\ @<<
	$<
<<

{release}.cxx{release\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Forelease\ @<<
	$<
<<

{release}.c{release\}.obj::
	$(CC) -c $(CFLAGS) $(INCPATH) -Forelease\ @<<
	$<
<<

{.}.cpp{release\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Forelease\ @<<
	$<
<<

{.}.cc{release\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Forelease\ @<<
	$<
<<

{.}.cxx{release\}.obj::
	$(CXX) -c $(CXXFLAGS) $(INCPATH) -Forelease\ @<<
	$<
<<

{.}.c{release\}.obj::
	$(CC) -c $(CFLAGS) $(INCPATH) -Forelease\ @<<
	$<
<<

####### Build rules

first: all
all: Makefile.Release  release\texturesurface.exe

release\texturesurface.exe: D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgets.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualization.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Gui.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Core.lib D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmain.lib $(OBJECTS) 
	$(LINKER) $(LFLAGS) /MANIFEST:embed /OUT:$(DESTDIR_TARGET) @<<
release\main.obj release\surfacegraph.obj release\topographicseries.obj release\highlightseries.obj release\custominputhandler.obj release\qrc_texturedsurface.obj release\moc_surfacegraph.obj release\moc_topographicseries.obj release\moc_highlightseries.obj release\moc_custominputhandler.obj
$(LIBS)
<<

qmake: FORCE
	@$(QMAKE) -o Makefile.Release texturesurface.pro -spec win32-msvc "CONFIG+=debug" "CONFIG+=qml_debug"

qmake_all: FORCE

dist:
	$(ZIP) texturesurface.zip $(SOURCES) $(DIST) texturesurface.pro ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\spec_pre.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\angle.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\windows-desktop.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\windows_vulkan_sdk.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\windows-vulkan.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\msvc-desktop.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\qconfig.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3danimation.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3danimation_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dcore.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dcore_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dextras.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dextras_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dinput.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dinput_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dlogic.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dlogic_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquick.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquick_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickanimation.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickanimation_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickextras.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickextras_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickinput.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickinput_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickrender.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickrender_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickscene2d.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3dquickscene2d_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3drender.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_3drender_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_accessibility_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axbase.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axbase_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axcontainer.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axcontainer_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axserver.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_axserver_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bluetooth.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bluetooth_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_bootstrap_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_charts.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_charts_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_concurrent.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_concurrent_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_core.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_core_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_datavisualization.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_datavisualization_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_dbus.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_dbus_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designer.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designer_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_designercomponents_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_devicediscovery_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_edid_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_egl_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_eventdispatcher_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_fb_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_fontdatabase_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gamepad.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gamepad_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gui.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_gui_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_help.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_help_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_location.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_location_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimedia.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimedia_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimediawidgets.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_multimediawidgets_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_network.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_network_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_networkauth.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_networkauth_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_nfc.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_nfc_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_opengl.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_opengl_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_openglextensions.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_openglextensions_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_packetprotocol_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_platformcompositor_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioning.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioning_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioningquick.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_positioningquick_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_printsupport.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_printsupport_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qml.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qml_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmldebug_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmldevtools_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlmodels.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlmodels_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmltest.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmltest_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlworkerscript.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qmlworkerscript_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_qtmultimediaquicktools_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quick.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quick_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickcontrols2.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickcontrols2_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickparticles_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickshapes_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quicktemplates2.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quicktemplates2_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickwidgets.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_quickwidgets_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_remoteobjects.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_remoteobjects_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_repparser.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_repparser_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_scxml.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_scxml_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sensors.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sensors_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialbus.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialbus_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialport.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_serialport_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sql.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_sql_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_svg.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_svg_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_testlib.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_testlib_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_texttospeech.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_texttospeech_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_theme_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uiplugin.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uitools.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_uitools_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_virtualkeyboard.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_virtualkeyboard_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_vulkan_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webchannel.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webchannel_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_websockets.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_websockets_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webview.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_webview_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_widgets.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_widgets_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_windowsuiautomation_support_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_winextras.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_winextras_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xml.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xml_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xmlpatterns.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_xmlpatterns_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\modules\qt_lib_zlib_private.pri ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt_functions.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt_config.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\win32-msvc\qmake.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\spec_post.prf .qmake.stash ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exclusive_builds.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\common\msvc-version.conf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\toolchain.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\default_pre.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\default_pre.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resolve_config.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exclusive_builds_post.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\default_post.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\build_pass.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qml_debug.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\precompile_header.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\warn_on.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qt.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resources_functions.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\resources.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\moc.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\opengl.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\uic.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\qmake_use.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\file_copies.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\win32\windows.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\testcase_targets.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\exceptions.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\yacc.prf ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\lex.prf texturesurface.pro texturedsurface.qrc ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Widgets.prl ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5DataVisualization.prl ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Gui.prl ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\Qt5Core.prl ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\lib\qtmain.prl   texturedsurface.qrc ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\data\dummy.cpp surfacegraph.h topographicseries.h highlightseries.h custominputhandler.h  main.cpp surfacegraph.cpp topographicseries.cpp highlightseries.cpp custominputhandler.cpp     

clean: compiler_clean 
	-$(DEL_FILE) release\main.obj release\surfacegraph.obj release\topographicseries.obj release\highlightseries.obj release\custominputhandler.obj release\qrc_texturedsurface.obj release\moc_surfacegraph.obj release\moc_topographicseries.obj release\moc_highlightseries.obj release\moc_custominputhandler.obj

distclean: clean 
	-$(DEL_FILE) .qmake.stash
	-$(DEL_FILE) $(DESTDIR_TARGET)
	-$(DEL_FILE) Makefile.Release

mocclean: compiler_moc_header_clean compiler_moc_objc_header_clean compiler_moc_source_clean

mocables: compiler_moc_header_make_all compiler_moc_objc_header_make_all compiler_moc_source_make_all

check: first

benchmark: first

compiler_no_pch_compiler_make_all:
compiler_no_pch_compiler_clean:
compiler_rcc_make_all: release\qrc_texturedsurface.cpp
compiler_rcc_clean:
	-$(DEL_FILE) release\qrc_texturedsurface.cpp
release\qrc_texturedsurface.cpp: texturedsurface.qrc \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\rcc.exe \
		topography.png \
		maptexture.jpg
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\rcc.exe -name texturedsurface texturedsurface.qrc -o release\qrc_texturedsurface.cpp

compiler_moc_predefs_make_all: release\moc_predefs.h
compiler_moc_predefs_clean:
	-$(DEL_FILE) release\moc_predefs.h
release\moc_predefs.h: ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\data\dummy.cpp
	cl -BxD:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\qmake.exe -nologo -Zc:wchar_t -FS -Zc:rvalueCast -Zc:inline -Zc:strictStrings -Zc:throwingNew -Zc:referenceBinding -Zc:__cplusplus -O2 -MD -W3 -w34100 -w34189 -w44996 -w44456 -w44457 -w44458 -wd4577 -wd4467 -E ..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\mkspecs\features\data\dummy.cpp 2>NUL >release\moc_predefs.h

compiler_moc_header_make_all: release\moc_surfacegraph.cpp release\moc_topographicseries.cpp release\moc_highlightseries.cpp release\moc_custominputhandler.cpp
compiler_moc_header_clean:
	-$(DEL_FILE) release\moc_surfacegraph.cpp release\moc_topographicseries.cpp release\moc_highlightseries.cpp release\moc_custominputhandler.cpp
release\moc_surfacegraph.cpp: surfacegraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DSurface \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QSlider \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgetsglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgets-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qabstractslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qwidget.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpalette.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontmetrics.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qsizepolicy.h \
		topographicseries.h \
		highlightseries.h \
		custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		release\moc_predefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe $(DEFINES) --compiler-flavor=msvc --include D:/PCLcode/texturesurface/texturesurface/release/moc_predefs.h -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/mkspecs/win32-msvc -ID:/PCLcode/texturesurface/texturesurface -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtWidgets -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtDataVisualization -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtGui -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtANGLE -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtCore -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\ATLMFC\include" -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\include" -I"C:\Program Files (x86)\Windows Kits\NETFXSDK\4.6.1\include\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\ucrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\shared" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\winrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\cppwinrt" surfacegraph.h -o release\moc_surfacegraph.cpp

release\moc_topographicseries.cpp: topographicseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		release\moc_predefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe $(DEFINES) --compiler-flavor=msvc --include D:/PCLcode/texturesurface/texturesurface/release/moc_predefs.h -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/mkspecs/win32-msvc -ID:/PCLcode/texturesurface/texturesurface -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtWidgets -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtDataVisualization -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtGui -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtANGLE -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtCore -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\ATLMFC\include" -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\include" -I"C:\Program Files (x86)\Windows Kits\NETFXSDK\4.6.1\include\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\ucrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\shared" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\winrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\cppwinrt" topographicseries.h -o release\moc_topographicseries.cpp

release\moc_highlightseries.cpp: highlightseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		topographicseries.h \
		release\moc_predefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe $(DEFINES) --compiler-flavor=msvc --include D:/PCLcode/texturesurface/texturesurface/release/moc_predefs.h -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/mkspecs/win32-msvc -ID:/PCLcode/texturesurface/texturesurface -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtWidgets -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtDataVisualization -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtGui -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtANGLE -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtCore -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\ATLMFC\include" -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\include" -I"C:\Program Files (x86)\Windows Kits\NETFXSDK\4.6.1\include\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\ucrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\shared" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\winrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\cppwinrt" highlightseries.h -o release\moc_highlightseries.cpp

release\moc_custominputhandler.cpp: custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		highlightseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		topographicseries.h \
		release\moc_predefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe
	D:\Qt\Qt5.14.2\5.14.2\msvc2017_64\bin\moc.exe $(DEFINES) --compiler-flavor=msvc --include D:/PCLcode/texturesurface/texturesurface/release/moc_predefs.h -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/mkspecs/win32-msvc -ID:/PCLcode/texturesurface/texturesurface -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtWidgets -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtDataVisualization -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtGui -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtANGLE -ID:/Qt/Qt5.14.2/5.14.2/msvc2017_64/include/QtCore -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\ATLMFC\include" -I"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\include" -I"C:\Program Files (x86)\Windows Kits\NETFXSDK\4.6.1\include\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\ucrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\shared" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\um" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\winrt" -I"C:\Program Files (x86)\Windows Kits\10\include\10.0.20348.0\cppwinrt" custominputhandler.h -o release\moc_custominputhandler.cpp

compiler_moc_objc_header_make_all:
compiler_moc_objc_header_clean:
compiler_moc_source_make_all:
compiler_moc_source_clean:
compiler_uic_make_all:
compiler_uic_clean:
compiler_yacc_decl_make_all:
compiler_yacc_decl_clean:
compiler_yacc_impl_make_all:
compiler_yacc_impl_clean:
compiler_lex_make_all:
compiler_lex_clean:
compiler_clean: compiler_rcc_clean compiler_moc_predefs_clean compiler_moc_header_clean 



####### Compile

release\main.obj: main.cpp surfacegraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DSurface \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QSlider \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgetsglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgets-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qabstractslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qwidget.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpalette.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontmetrics.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qsizepolicy.h \
		topographicseries.h \
		highlightseries.h \
		custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QApplication \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qapplication.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreapplication.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qeventloop.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qdesktopwidget.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qguiapplication.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qinputmethod.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QWidget \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QHBoxLayout \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qboxlayout.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qlayout.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qlayoutitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qgridlayout.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QVBoxLayout \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QGroupBox \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qgroupbox.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qframe.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QCheckBox \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qcheckbox.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qabstractbutton.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QLabel \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qlabel.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QMessageBox \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qmessagebox.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qdialog.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QScreen \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qscreen.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QSize \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QSizeF \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTransform \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QPainter \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtextoption.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpen.h

release\surfacegraph.obj: surfacegraph.cpp surfacegraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DSurface \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\QSlider \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgetsglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qtwidgets-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qabstractslider.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qwidget.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpalette.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontmetrics.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfontinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtWidgets\qsizepolicy.h \
		topographicseries.h \
		highlightseries.h \
		custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DTheme

release\topographicseries.obj: topographicseries.cpp topographicseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D

release\highlightseries.obj: highlightseries.cpp highlightseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		topographicseries.h

release\custominputhandler.obj: custominputhandler.cpp custominputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DInputHandler \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dinputhandler.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qdatavisualizationglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig-bootstrapped.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qconfig.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtcore-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsystemdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qprocessordetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcompilerdetection.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtypeinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsysinfo.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlogging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qflags.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbasicatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_bootstrap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qgenericatomic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_cxx11.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qatomic_msvc.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qglobalstatic.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmutex.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnumeric.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qversiontagging.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dscene.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dcamera.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QObject \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qnamespace.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobjectdefs_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstring.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qchar.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrefcount.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qarraydata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringliteral.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringview.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringbuilder.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qalgorithms.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiterator.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhashfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpair.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvector.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainertools_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qpoint.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qbytearraylist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringlist.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qregexp.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qstringmatcher.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcoreevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qscopedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmetatype.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvarlengtharray.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontainerfwd.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qobject_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QVector3D \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector3d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtguiglobal.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtgui-config.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dlight.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QRect \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qrect.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmargins.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsize.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QPoint \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWheelEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qevent.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindowdefs_win.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qregion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdatastream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qiodevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qkeysequence.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qvariant.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qdebug.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qhash.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qtextstream.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qlocale.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qshareddata.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qset.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qcontiguouscache.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qsharedpointer_impl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qurlquery.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfile.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qfiledevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector2d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtouchdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QMouseEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QTouchEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QAbstract3DGraph \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dgraph.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\q3dtheme.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QLinearGradient \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qbrush.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcolor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgb.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qrgba64.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpolygon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qline.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qtransform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpainterpath.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qimage.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpaintdevice.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixelformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qpixmap.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QFont \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qfont.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QColor \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QWindow \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qwindow.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QEvent \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QMargins \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurface.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qsurfaceformat.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qicon.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qcursor.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QOpenGLFunctions \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengl.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qt_windows.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES3\gl3platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\KHR\khrplatform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtANGLE\GLES2\gl2platform.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopengles2ext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglcontext.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QScopedPointer \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QSurfaceFormat \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qopenglversionfunctions.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QLocale \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QValue3DAxis \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3daxis.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QStringList \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qvalue3daxisformatter.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QVector \
		highlightseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\QSurface3DSeries \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurface3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstract3dseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\QString \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\QQuaternion \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qquaternion.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qgenericmatrix.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtGui\qvector4d.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qabstractdataproxy.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\qsurfacedataitem.h \
		topographicseries.h \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtDataVisualization\Q3DCamera \
		..\..\..\Qt\Qt5.14.2\5.14.2\msvc2017_64\include\QtCore\qmath.h

release\qrc_texturedsurface.obj: release\qrc_texturedsurface.cpp 

release\moc_surfacegraph.obj: release\moc_surfacegraph.cpp 

release\moc_topographicseries.obj: release\moc_topographicseries.cpp 

release\moc_highlightseries.obj: release\moc_highlightseries.cpp 

release\moc_custominputhandler.obj: release\moc_custominputhandler.cpp 

####### Install

install:  FORCE

uninstall:  FORCE

FORCE:


/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#include "surfacegraph.h"
#include "topographicseries.h"

#include <QtDataVisualization/QValue3DAxis>
#include <QtDataVisualization/Q3DTheme>

const float areaWidth = 8000.0f;
const float areaHeight = 8000.0f;
const float aspectRatio = 0.1389f;
const float minRange = areaWidth * 0.49f;

SurfaceGraph::SurfaceGraph(Q3DSurface *surface)
    : m_graph(surface)
{
    m_graph->setAxisX(new QValue3DAxis);
    m_graph->setAxisY(new QValue3DAxis);
    m_graph->setAxisZ(new QValue3DAxis);
    m_graph->axisX()->setLabelFormat("%i");
    m_graph->axisZ()->setLabelFormat("%i");
    m_graph->axisX()->setRange(0.0f, areaWidth);
    m_graph->axisY()->setRange(100.0f, areaWidth * aspectRatio);
    m_graph->axisZ()->setRange(0.0f, areaHeight);
    m_graph->axisX()->setLabelAutoRotation(30);
    m_graph->axisY()->setLabelAutoRotation(90);
    m_graph->axisZ()->setLabelAutoRotation(30);
    m_graph->activeTheme()->setType(Q3DTheme::ThemePrimaryColors);

    QFont font = m_graph->activeTheme()->font();
    font.setPointSize(20);
    m_graph->activeTheme()->setFont(font);

    m_topography = new TopographicSeries();
    m_topography->setTopographyFile(":/maps/topography", areaWidth, areaHeight);
    m_topography->setItemLabelFormat(QStringLiteral("@yLabel m"));

    m_highlight = new HighlightSeries();
    m_highlight->setTopographicSeries(m_topography);
    m_highlight->setMinHeight(minRange * aspectRatio);
    m_highlight->handleGradientChange(areaWidth * aspectRatio);
//! [1]
    QObject::connect(m_graph->axisY(), &QValue3DAxis::maxChanged,
                     m_highlight, &HighlightSeries::handleGradientChange);
//! [1]

    m_graph->addSeries(m_topography);
    m_graph->addSeries(m_highlight);

    m_inputHandler = new CustomInputHandler(m_graph);
    m_inputHandler->setHighlightSeries(m_highlight);
    m_inputHandler->setAxes(m_graph->axisX(), m_graph->axisY(), m_graph->axisZ());
    m_inputHandler->setLimits(0.0f, areaWidth, minRange);
    m_inputHandler->setAspectRatio(aspectRatio);

    m_graph->setActiveInputHandler(m_inputHandler);
}

SurfaceGraph::~SurfaceGraph()
{
    delete m_graph;
}

//! [0]
void SurfaceGraph::toggleSurfaceTexture(bool enable)
{
    if (enable)
        m_topography->setTextureFile(":/maps/maptexture");
    else
        m_topography->setTextureFile("");
}
//! [0]

/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#ifndef SURFACEGRAPH_H
#define SURFACEGRAPH_H

#include <QtDataVisualization/Q3DSurface>
#include <QtDataVisualization/QSurface3DSeries>
#include <QtWidgets/QSlider>
#include "topographicseries.h"
#include "highlightseries.h"
#include "custominputhandler.h"

class SurfaceGraph : public QObject
{
    Q_OBJECT
public:
    explicit SurfaceGraph(Q3DSurface *surface);
    ~SurfaceGraph();

    void toggleSurfaceTexture(bool enable);

private:
    Q3DSurface *m_graph;

    TopographicSeries *m_topography;
    HighlightSeries *m_highlight;
    int m_highlightWidth;
    int m_highlightHeight;

    CustomInputHandler *m_inputHandler;
};

#endif // SURFACEGRAPH_H

<RCC>
    <qresource prefix="/maps">
        <file alias="topography">topography.png</file>
        <file alias="maptexture">maptexture.jpg</file>
    </qresource>
</RCC>



SOURCES += main.cpp \
           surfacegraph.cpp \
           topographicseries.cpp \
           highlightseries.cpp \
           custominputhandler.cpp

HEADERS += surfacegraph.h \
           topographicseries.h \
           highlightseries.h \
           custominputhandler.h

QT += widgets
QT += datavisualization
RESOURCES += texturedsurface.qrc

OTHER_FILES += doc/src/* \
               doc/images/*

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE QtCreatorProject>
<!-- Written by QtCreator 19.0.0, 2026-06-12T10:03:41. -->
<qtcreator>
 <data>
  <variable>EnvironmentId</variable>
  <value type="QByteArray">{71f7b2ef-1f8f-4e6b-881f-abba7be351d0}</value>
 </data>
 <data>
  <variable>ProjectExplorer.Project.ActiveTarget</variable>
  <value type="qlonglong">0</value>
 </data>
 <data>
  <variable>ProjectExplorer.Project.EditorSettings</variable>
  <valuemap type="QVariantMap">
   <value type="bool" key="EditorConfiguration.AutoDetect">true</value>
   <value type="bool" key="EditorConfiguration.AutoIndent">true</value>
   <value type="bool" key="EditorConfiguration.CamelCaseNavigation">true</value>
   <valuemap type="QVariantMap" key="EditorConfiguration.CodeStyle.0">
    <value type="QString" key="language">Cpp</value>
    <valuemap type="QVariantMap" key="value">
     <value type="QByteArray" key="CurrentPreferences">CppGlobal</value>
    </valuemap>
   </valuemap>
   <valuemap type="QVariantMap" key="EditorConfiguration.CodeStyle.1">
    <value type="QString" key="language">QmlJS</value>
    <valuemap type="QVariantMap" key="value">
     <value type="QByteArray" key="CurrentPreferences">QmlJSGlobal</value>
    </valuemap>
   </valuemap>
   <value type="qlonglong" key="EditorConfiguration.CodeStyle.Count">2</value>
   <value type="QByteArray" key="EditorConfiguration.Codec">UTF-8</value>
   <value type="bool" key="EditorConfiguration.ConstrainTooltips">false</value>
   <value type="int" key="EditorConfiguration.IndentSize">4</value>
   <value type="bool" key="EditorConfiguration.KeyboardTooltips">false</value>
   <value type="int" key="EditorConfiguration.LineEndingBehavior">0</value>
   <value type="int" key="EditorConfiguration.MarginColumn">80</value>
   <value type="bool" key="EditorConfiguration.MouseHiding">true</value>
   <value type="bool" key="EditorConfiguration.MouseNavigation">true</value>
   <value type="int" key="EditorConfiguration.PaddingMode">1</value>
   <value type="int" key="EditorConfiguration.PreferAfterWhitespaceComments">0</value>
   <value type="bool" key="EditorConfiguration.PreferSingleLineComments">false</value>
   <value type="bool" key="EditorConfiguration.ScrollWheelZooming">true</value>
   <value type="bool" key="EditorConfiguration.ShowMargin">false</value>
   <value type="int" key="EditorConfiguration.SmartBackspaceBehavior">2</value>
   <value type="bool" key="EditorConfiguration.SmartSelectionChanging">true</value>
   <value type="bool" key="EditorConfiguration.SpacesForTabs">true</value>
   <value type="int" key="EditorConfiguration.TabKeyBehavior">0</value>
   <value type="int" key="EditorConfiguration.TabSize">8</value>
   <value type="bool" key="EditorConfiguration.UseGlobal">true</value>
   <value type="bool" key="EditorConfiguration.UseIndenter">false</value>
   <value type="int" key="EditorConfiguration.Utf8BomBehavior">1</value>
   <value type="bool" key="EditorConfiguration.addFinalNewLine">false</value>
   <value type="bool" key="EditorConfiguration.cleanIndentation">true</value>
   <value type="bool" key="EditorConfiguration.cleanWhitespace">true</value>
   <value type="QString" key="EditorConfiguration.ignoreFileTypes">*.md, *.MD, Makefile</value>
   <value type="bool" key="EditorConfiguration.inEntireDocument">false</value>
   <value type="bool" key="EditorConfiguration.skipTrailingWhitespace">true</value>
   <value type="bool" key="EditorConfiguration.tintMarginArea">true</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.PluginSettings</variable>
  <valuemap type="QVariantMap">
   <valuemap type="QVariantMap" key="AutoTest.ActiveFrameworks">
    <value type="bool" key="AutoTest.Framework.Boost">true</value>
    <value type="bool" key="AutoTest.Framework.CTest">false</value>
    <value type="bool" key="AutoTest.Framework.Catch">true</value>
    <value type="bool" key="AutoTest.Framework.GTest">true</value>
    <value type="bool" key="AutoTest.Framework.QtQuickTest">true</value>
    <value type="bool" key="AutoTest.Framework.QtTest">true</value>
   </valuemap>
   <value type="bool" key="AutoTest.ApplyFilter">false</value>
   <valuemap type="QVariantMap" key="AutoTest.CheckStates"/>
   <valuelist type="QVariantList" key="AutoTest.PathFilters"/>
   <value type="int" key="AutoTest.RunAfterBuild">0</value>
   <value type="bool" key="AutoTest.UseGlobal">true</value>
   <valuemap type="QVariantMap" key="ClangTools">
    <value type="bool" key="ClangTools.AnalyzeOpenFiles">true</value>
    <value type="bool" key="ClangTools.BuildBeforeAnalysis">true</value>
    <value type="QString" key="ClangTools.DiagnosticConfig">Builtin.DefaultTidyAndClazy</value>
    <value type="int" key="ClangTools.ParallelJobs">16</value>
    <value type="bool" key="ClangTools.PreferConfigFile">true</value>
    <valuelist type="QVariantList" key="ClangTools.SelectedDirs"/>
    <valuelist type="QVariantList" key="ClangTools.SelectedFiles"/>
    <valuelist type="QVariantList" key="ClangTools.SuppressedDiagnostics"/>
    <value type="bool" key="ClangTools.UseGlobalSettings">true</value>
   </valuemap>
   <value type="int" key="RcSync">0</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.Target.0</variable>
  <valuemap type="QVariantMap">
   <value type="QString" key="DeviceType">Desktop</value>
   <value type="bool" key="HasPerBcDcs">true</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Desktop Qt 6.11.0 MSVC2022 64bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Desktop Qt 6.11.0 MSVC2022 64bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">qt.qt6.6110.win64_msvc2022_64_kit</value>
   <value type="qlonglong" key="ProjectExplorer.Target.ActiveBuildConfiguration">0</value>
   <value type="qlonglong" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
   <value type="qlonglong" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.0">
    <value type="int" key="EnableQmlDebugging">0</value>
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">.\build\%{Asciify:%{Kit:FileSystemName}-%{BuildConfig:Name}}</value>
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory.shadowDir">build/Desktop_Qt_6_11_0_MSVC2022_64bit-Debug</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <valuelist type="QVariantList" key="QtProjectManager.QMakeBuildStep.SelectedAbis"/>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
     </valuemap>
     <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">构建</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">构建</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
     </valuemap>
     <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">清除</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">清除</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.CustomParsers"/>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ParseStandardOutput">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Debug</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="qlonglong" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
    <value type="qlonglong" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
      <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">部署</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">部署</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
     <valuemap type="QVariantMap" key="ProjectExplorer.DeployConfiguration.CustomData"/>
     <value type="bool" key="ProjectExplorer.DeployConfiguration.CustomDataEnabled">false</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
    </valuemap>
    <value type="qlonglong" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
     <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
     <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
     <value type="int" key="Analyzer.Valgrind.Callgrind.CostFormat">0</value>
     <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
     <valuelist type="QVariantList" key="Analyzer.Valgrind.SuppressionFiles"/>
     <valuelist type="QVariantList" key="CustomOutputParsers"/>
     <value type="int" key="PE.EnvironmentAspect.Base">2</value>
     <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
     <value type="bool" key="PE.EnvironmentAspect.PrintOnRun">false</value>
     <value type="QString" key="PerfRecordArgsId">-e cpu-cycles --call-graph &quot;dwarf,4096&quot; -F 250</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName"></value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4RunConfiguration:</value>
     <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey">E:/software/texturesurface/texturesurface.pro</value>
     <value type="bool" key="ProjectExplorer.RunConfiguration.Customized">false</value>
     <value type="QString" key="ProjectExplorer.RunConfiguration.UniqueId"></value>
     <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
     <value type="bool" key="RunConfiguration.UseLibrarySearchPath">true</value>
     <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
     <value type="QString" key="RunConfiguration.WorkingDirectory.default">%{RunConfig:Executable:Path}</value>
    </valuemap>
    <value type="qlonglong" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">2</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.1">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">.\build\%{Asciify:%{Kit:FileSystemName}-%{BuildConfig:Name}}</value>
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory.shadowDir">build/Desktop_Qt_6_11_0_MSVC2022_64bit-Release</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <valuelist type="QVariantList" key="QtProjectManager.QMakeBuildStep.SelectedAbis"/>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
     </valuemap>
     <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">构建</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">构建</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
     </valuemap>
     <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">清除</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">清除</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.CustomParsers"/>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ParseStandardOutput">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Release</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="qlonglong" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
    <value type="qlonglong" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
      <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">部署</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">部署</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
     <valuemap type="QVariantMap" key="ProjectExplorer.DeployConfiguration.CustomData"/>
     <value type="bool" key="ProjectExplorer.DeployConfiguration.CustomDataEnabled">false</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
    </valuemap>
    <value type="qlonglong" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
     <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
     <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
     <value type="int" key="Analyzer.Valgrind.Callgrind.CostFormat">0</value>
     <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
     <valuelist type="QVariantList" key="Analyzer.Valgrind.SuppressionFiles"/>
     <valuelist type="QVariantList" key="CustomOutputParsers"/>
     <value type="int" key="PE.EnvironmentAspect.Base">2</value>
     <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
     <value type="bool" key="PE.EnvironmentAspect.PrintOnRun">false</value>
     <value type="QString" key="PerfRecordArgsId">-e cpu-cycles --call-graph &quot;dwarf,4096&quot; -F 250</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName"></value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.CustomExecutableRunConfiguration</value>
     <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey"></value>
     <value type="bool" key="ProjectExplorer.RunConfiguration.Customized">false</value>
     <value type="QString" key="ProjectExplorer.RunConfiguration.UniqueId"></value>
     <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
     <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
     <value type="QString" key="RunConfiguration.WorkingDirectory.default">%{RunConfig:Executable:Path}</value>
    </valuemap>
    <value type="qlonglong" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
    <value type="int" key="QtQuickCompiler">0</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.2">
    <value type="int" key="EnableQmlDebugging">0</value>
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">.\build\%{Asciify:%{Kit:FileSystemName}-%{BuildConfig:Name}}</value>
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory.shadowDir">build/Desktop_Qt_6_11_0_MSVC2022_64bit-Profile</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <valuelist type="QVariantList" key="QtProjectManager.QMakeBuildStep.SelectedAbis"/>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
     </valuemap>
     <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">构建</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">构建</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
     </valuemap>
     <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">清除</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">清除</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.CustomParsers"/>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ParseStandardOutput">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Profile</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="qlonglong" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
    <value type="qlonglong" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
      <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">部署</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">部署</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
     <valuemap type="QVariantMap" key="ProjectExplorer.DeployConfiguration.CustomData"/>
     <value type="bool" key="ProjectExplorer.DeployConfiguration.CustomDataEnabled">false</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
    </valuemap>
    <value type="qlonglong" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
     <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
     <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
     <value type="int" key="Analyzer.Valgrind.Callgrind.CostFormat">0</value>
     <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
     <valuelist type="QVariantList" key="Analyzer.Valgrind.SuppressionFiles"/>
     <valuelist type="QVariantList" key="CustomOutputParsers"/>
     <value type="int" key="PE.EnvironmentAspect.Base">2</value>
     <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
     <value type="bool" key="PE.EnvironmentAspect.PrintOnRun">false</value>
     <value type="QString" key="PerfRecordArgsId">-e cpu-cycles --call-graph &quot;dwarf,4096&quot; -F 250</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName"></value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.CustomExecutableRunConfiguration</value>
     <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey"></value>
     <value type="bool" key="ProjectExplorer.RunConfiguration.Customized">false</value>
     <value type="QString" key="ProjectExplorer.RunConfiguration.UniqueId"></value>
     <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
     <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
     <value type="QString" key="RunConfiguration.WorkingDirectory.default">%{RunConfig:Executable:Path}</value>
    </valuemap>
    <value type="qlonglong" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
    <value type="int" key="QtQuickCompiler">0</value>
    <value type="int" key="SeparateDebugInfo">0</value>
   </valuemap>
   <value type="qlonglong" key="ProjectExplorer.Target.BuildConfigurationCount">3</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <value type="qlonglong" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">部署</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">部署</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.DeployConfiguration.CustomData"/>
    <value type="bool" key="ProjectExplorer.DeployConfiguration.CustomDataEnabled">false</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
   </valuemap>
   <value type="qlonglong" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
    <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
    <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
    <value type="int" key="Analyzer.Valgrind.Callgrind.CostFormat">0</value>
    <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.SuppressionFiles"/>
    <valuelist type="QVariantList" key="CustomOutputParsers"/>
    <value type="int" key="PE.EnvironmentAspect.Base">2</value>
    <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
    <value type="bool" key="PE.EnvironmentAspect.PrintOnRun">false</value>
    <value type="QString" key="PerfRecordArgsId">-e cpu-cycles --call-graph &quot;dwarf,4096&quot; -F 250</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName"></value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4RunConfiguration:</value>
    <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey">E:/software/texturesurface/texturesurface.pro</value>
    <value type="bool" key="ProjectExplorer.RunConfiguration.Customized">false</value>
    <value type="QString" key="ProjectExplorer.RunConfiguration.UniqueId"></value>
    <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
    <value type="bool" key="RunConfiguration.UseLibrarySearchPath">true</value>
    <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
    <value type="QString" key="RunConfiguration.WorkingDirectory.default">%{RunConfig:Executable:Path}</value>
   </valuemap>
   <value type="qlonglong" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.TargetCount</variable>
  <value type="qlonglong">1</value>
 </data>
 <data>
  <variable>Version</variable>
  <value type="int">22</value>
 </data>
</qtcreator>

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE QtCreatorProject>
<!-- Written by QtCreator 4.11.1, 2023-09-20T22:13:01. -->
<qtcreator>
 <data>
  <variable>EnvironmentId</variable>
  <value type="QByteArray">{9dab3cb6-2451-428d-9461-61d762140b95}</value>
 </data>
 <data>
  <variable>ProjectExplorer.Project.ActiveTarget</variable>
  <value type="int">0</value>
 </data>
 <data>
  <variable>ProjectExplorer.Project.EditorSettings</variable>
  <valuemap type="QVariantMap">
   <value type="bool" key="EditorConfiguration.AutoIndent">true</value>
   <value type="bool" key="EditorConfiguration.AutoSpacesForTabs">false</value>
   <value type="bool" key="EditorConfiguration.CamelCaseNavigation">true</value>
   <valuemap type="QVariantMap" key="EditorConfiguration.CodeStyle.0">
    <value type="QString" key="language">Cpp</value>
    <valuemap type="QVariantMap" key="value">
     <value type="QByteArray" key="CurrentPreferences">CppGlobal</value>
    </valuemap>
   </valuemap>
   <valuemap type="QVariantMap" key="EditorConfiguration.CodeStyle.1">
    <value type="QString" key="language">QmlJS</value>
    <valuemap type="QVariantMap" key="value">
     <value type="QByteArray" key="CurrentPreferences">QmlJSGlobal</value>
    </valuemap>
   </valuemap>
   <value type="int" key="EditorConfiguration.CodeStyle.Count">2</value>
   <value type="QByteArray" key="EditorConfiguration.Codec">UTF-8</value>
   <value type="bool" key="EditorConfiguration.ConstrainTooltips">false</value>
   <value type="int" key="EditorConfiguration.IndentSize">4</value>
   <value type="bool" key="EditorConfiguration.KeyboardTooltips">false</value>
   <value type="int" key="EditorConfiguration.MarginColumn">80</value>
   <value type="bool" key="EditorConfiguration.MouseHiding">true</value>
   <value type="bool" key="EditorConfiguration.MouseNavigation">true</value>
   <value type="int" key="EditorConfiguration.PaddingMode">1</value>
   <value type="bool" key="EditorConfiguration.ScrollWheelZooming">true</value>
   <value type="bool" key="EditorConfiguration.ShowMargin">false</value>
   <value type="int" key="EditorConfiguration.SmartBackspaceBehavior">0</value>
   <value type="bool" key="EditorConfiguration.SmartSelectionChanging">true</value>
   <value type="bool" key="EditorConfiguration.SpacesForTabs">true</value>
   <value type="int" key="EditorConfiguration.TabKeyBehavior">0</value>
   <value type="int" key="EditorConfiguration.TabSize">8</value>
   <value type="bool" key="EditorConfiguration.UseGlobal">true</value>
   <value type="int" key="EditorConfiguration.Utf8BomBehavior">1</value>
   <value type="bool" key="EditorConfiguration.addFinalNewLine">true</value>
   <value type="bool" key="EditorConfiguration.cleanIndentation">true</value>
   <value type="bool" key="EditorConfiguration.cleanWhitespace">true</value>
   <value type="bool" key="EditorConfiguration.inEntireDocument">false</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.PluginSettings</variable>
  <valuemap type="QVariantMap">
   <valuelist type="QVariantList" key="ClangCodeModel.CustomCommandLineKey">
    <value type="QString">-fno-delayed-template-parsing</value>
   </valuelist>
   <value type="bool" key="ClangCodeModel.UseGlobalConfig">true</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.Target.0</variable>
  <valuemap type="QVariantMap">
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Desktop Qt 5.14.2 MSVC2017 32bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Desktop Qt 5.14.2 MSVC2017 32bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">qt.qt5.5142.win32_msvc2017_kit</value>
   <value type="int" key="ProjectExplorer.Target.ActiveBuildConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.0">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_32bit-Debug</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">false</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Debug</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">2</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.1">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_32bit-Release</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">false</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Release</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.2">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_32bit-Profile</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">true</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Profile</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.BuildConfigurationCount">3</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.PluginSettings"/>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
    <value type="QString" key="Analyzer.Perf.CallgraphMode">dwarf</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.Events">
     <value type="QString">cpu-cycles</value>
    </valuelist>
    <valuelist type="QVariantList" key="Analyzer.Perf.ExtraArguments"/>
    <value type="int" key="Analyzer.Perf.Frequency">250</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.RecordArguments">
     <value type="QString">-e</value>
     <value type="QString">cpu-cycles</value>
     <value type="QString">--call-graph</value>
     <value type="QString">dwarf,4096</value>
     <value type="QString">-F</value>
     <value type="QString">250</value>
    </valuelist>
    <value type="QString" key="Analyzer.Perf.SampleMode">-F</value>
    <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
    <value type="int" key="Analyzer.Perf.StackSize">4096</value>
    <value type="bool" key="Analyzer.QmlProfiler.AggregateTraces">false</value>
    <value type="bool" key="Analyzer.QmlProfiler.FlushEnabled">false</value>
    <value type="uint" key="Analyzer.QmlProfiler.FlushInterval">1000</value>
    <value type="QString" key="Analyzer.QmlProfiler.LastTraceFile"></value>
    <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.AddedSuppressionFiles"/>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectBusEvents">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectSystime">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableBranchSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableCacheSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableEventToolTips">true</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.MinimumCostRatio">0.01</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.VisualisationMinimumCostRatio">10</value>
    <value type="bool" key="Analyzer.Valgrind.FilterExternalIssues">true</value>
    <value type="QString" key="Analyzer.Valgrind.KCachegrindExecutable">kcachegrind</value>
    <value type="int" key="Analyzer.Valgrind.LeakCheckOnFinish">1</value>
    <value type="int" key="Analyzer.Valgrind.NumCallers">25</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.RemovedSuppressionFiles"/>
    <value type="int" key="Analyzer.Valgrind.SelfModifyingCodeDetection">1</value>
    <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
    <value type="bool" key="Analyzer.Valgrind.ShowReachable">false</value>
    <value type="bool" key="Analyzer.Valgrind.TrackOrigins">true</value>
    <value type="QString" key="Analyzer.Valgrind.ValgrindExecutable">valgrind</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.VisibleErrorKinds">
     <value type="int">0</value>
     <value type="int">1</value>
     <value type="int">2</value>
     <value type="int">3</value>
     <value type="int">4</value>
     <value type="int">5</value>
     <value type="int">6</value>
     <value type="int">7</value>
     <value type="int">8</value>
     <value type="int">9</value>
     <value type="int">10</value>
     <value type="int">11</value>
     <value type="int">12</value>
     <value type="int">13</value>
     <value type="int">14</value>
    </valuelist>
    <value type="int" key="PE.EnvironmentAspect.Base">2</value>
    <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4RunConfiguration:D:/qtlai/texturesurface/texturesurface.pro</value>
    <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey">D:/qtlai/texturesurface/texturesurface.pro</value>
    <value type="QString" key="RunConfiguration.Arguments"></value>
    <value type="bool" key="RunConfiguration.Arguments.multi">false</value>
    <value type="QString" key="RunConfiguration.OverrideDebuggerStartup"></value>
    <value type="bool" key="RunConfiguration.UseCppDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
    <value type="bool" key="RunConfiguration.UseLibrarySearchPath">true</value>
    <value type="bool" key="RunConfiguration.UseMultiProcess">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
    <value type="QString" key="RunConfiguration.WorkingDirectory"></value>
    <value type="QString" key="RunConfiguration.WorkingDirectory.default">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_32bit-Debug</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.Target.1</variable>
  <valuemap type="QVariantMap">
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Desktop Qt 5.14.2 MSVC2017 64bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Desktop Qt 5.14.2 MSVC2017 64bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">qt.qt5.5142.win64_msvc2017_64_kit</value>
   <value type="int" key="ProjectExplorer.Target.ActiveBuildConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.0">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_64bit-Debug</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">false</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Debug</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">2</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.1">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_64bit-Release</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">false</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Release</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.2">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_64bit-Profile</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">true</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Profile</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.BuildConfigurationCount">3</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.PluginSettings"/>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
    <value type="QString" key="Analyzer.Perf.CallgraphMode">dwarf</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.Events">
     <value type="QString">cpu-cycles</value>
    </valuelist>
    <valuelist type="QVariantList" key="Analyzer.Perf.ExtraArguments"/>
    <value type="int" key="Analyzer.Perf.Frequency">250</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.RecordArguments">
     <value type="QString">-e</value>
     <value type="QString">cpu-cycles</value>
     <value type="QString">--call-graph</value>
     <value type="QString">dwarf,4096</value>
     <value type="QString">-F</value>
     <value type="QString">250</value>
    </valuelist>
    <value type="QString" key="Analyzer.Perf.SampleMode">-F</value>
    <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
    <value type="int" key="Analyzer.Perf.StackSize">4096</value>
    <value type="bool" key="Analyzer.QmlProfiler.AggregateTraces">false</value>
    <value type="bool" key="Analyzer.QmlProfiler.FlushEnabled">false</value>
    <value type="uint" key="Analyzer.QmlProfiler.FlushInterval">1000</value>
    <value type="QString" key="Analyzer.QmlProfiler.LastTraceFile"></value>
    <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.AddedSuppressionFiles"/>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectBusEvents">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectSystime">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableBranchSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableCacheSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableEventToolTips">true</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.MinimumCostRatio">0.01</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.VisualisationMinimumCostRatio">10</value>
    <value type="bool" key="Analyzer.Valgrind.FilterExternalIssues">true</value>
    <value type="QString" key="Analyzer.Valgrind.KCachegrindExecutable">kcachegrind</value>
    <value type="int" key="Analyzer.Valgrind.LeakCheckOnFinish">1</value>
    <value type="int" key="Analyzer.Valgrind.NumCallers">25</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.RemovedSuppressionFiles"/>
    <value type="int" key="Analyzer.Valgrind.SelfModifyingCodeDetection">1</value>
    <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
    <value type="bool" key="Analyzer.Valgrind.ShowReachable">false</value>
    <value type="bool" key="Analyzer.Valgrind.TrackOrigins">true</value>
    <value type="QString" key="Analyzer.Valgrind.ValgrindExecutable">valgrind</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.VisibleErrorKinds">
     <value type="int">0</value>
     <value type="int">1</value>
     <value type="int">2</value>
     <value type="int">3</value>
     <value type="int">4</value>
     <value type="int">5</value>
     <value type="int">6</value>
     <value type="int">7</value>
     <value type="int">8</value>
     <value type="int">9</value>
     <value type="int">10</value>
     <value type="int">11</value>
     <value type="int">12</value>
     <value type="int">13</value>
     <value type="int">14</value>
    </valuelist>
    <value type="int" key="PE.EnvironmentAspect.Base">2</value>
    <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
    <value type="QString" key="ProjectExplorer.CustomExecutableRunConfiguration.Executable"></value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.CustomExecutableRunConfiguration</value>
    <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey"></value>
    <value type="QString" key="RunConfiguration.Arguments"></value>
    <value type="bool" key="RunConfiguration.Arguments.multi">false</value>
    <value type="QString" key="RunConfiguration.OverrideDebuggerStartup"></value>
    <value type="bool" key="RunConfiguration.UseCppDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
    <value type="bool" key="RunConfiguration.UseMultiProcess">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
    <value type="QString" key="RunConfiguration.WorkingDirectory"></value>
    <value type="QString" key="RunConfiguration.WorkingDirectory.default"></value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.Target.2</variable>
  <valuemap type="QVariantMap">
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Desktop Qt 5.14.2 MinGW 32-bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Desktop Qt 5.14.2 MinGW 32-bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">qt.qt5.5142.win32_mingw73_kit</value>
   <value type="int" key="ProjectExplorer.Target.ActiveBuildConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.0">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MinGW_32_bit-Debug</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">false</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Debug</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">2</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.1">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MinGW_32_bit-Release</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">false</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Release</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.2">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MinGW_32_bit-Profile</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">true</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Profile</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.BuildConfigurationCount">3</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.PluginSettings"/>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
    <value type="QString" key="Analyzer.Perf.CallgraphMode">dwarf</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.Events">
     <value type="QString">cpu-cycles</value>
    </valuelist>
    <valuelist type="QVariantList" key="Analyzer.Perf.ExtraArguments"/>
    <value type="int" key="Analyzer.Perf.Frequency">250</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.RecordArguments">
     <value type="QString">-e</value>
     <value type="QString">cpu-cycles</value>
     <value type="QString">--call-graph</value>
     <value type="QString">dwarf,4096</value>
     <value type="QString">-F</value>
     <value type="QString">250</value>
    </valuelist>
    <value type="QString" key="Analyzer.Perf.SampleMode">-F</value>
    <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
    <value type="int" key="Analyzer.Perf.StackSize">4096</value>
    <value type="bool" key="Analyzer.QmlProfiler.AggregateTraces">false</value>
    <value type="bool" key="Analyzer.QmlProfiler.FlushEnabled">false</value>
    <value type="uint" key="Analyzer.QmlProfiler.FlushInterval">1000</value>
    <value type="QString" key="Analyzer.QmlProfiler.LastTraceFile"></value>
    <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.AddedSuppressionFiles"/>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectBusEvents">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectSystime">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableBranchSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableCacheSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableEventToolTips">true</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.MinimumCostRatio">0.01</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.VisualisationMinimumCostRatio">10</value>
    <value type="bool" key="Analyzer.Valgrind.FilterExternalIssues">true</value>
    <value type="QString" key="Analyzer.Valgrind.KCachegrindExecutable">kcachegrind</value>
    <value type="int" key="Analyzer.Valgrind.LeakCheckOnFinish">1</value>
    <value type="int" key="Analyzer.Valgrind.NumCallers">25</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.RemovedSuppressionFiles"/>
    <value type="int" key="Analyzer.Valgrind.SelfModifyingCodeDetection">1</value>
    <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
    <value type="bool" key="Analyzer.Valgrind.ShowReachable">false</value>
    <value type="bool" key="Analyzer.Valgrind.TrackOrigins">true</value>
    <value type="QString" key="Analyzer.Valgrind.ValgrindExecutable">valgrind</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.VisibleErrorKinds">
     <value type="int">0</value>
     <value type="int">1</value>
     <value type="int">2</value>
     <value type="int">3</value>
     <value type="int">4</value>
     <value type="int">5</value>
     <value type="int">6</value>
     <value type="int">7</value>
     <value type="int">8</value>
     <value type="int">9</value>
     <value type="int">10</value>
     <value type="int">11</value>
     <value type="int">12</value>
     <value type="int">13</value>
     <value type="int">14</value>
    </valuelist>
    <value type="int" key="PE.EnvironmentAspect.Base">2</value>
    <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
    <value type="QString" key="ProjectExplorer.CustomExecutableRunConfiguration.Executable"></value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.CustomExecutableRunConfiguration</value>
    <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey"></value>
    <value type="QString" key="RunConfiguration.Arguments"></value>
    <value type="bool" key="RunConfiguration.Arguments.multi">false</value>
    <value type="QString" key="RunConfiguration.OverrideDebuggerStartup"></value>
    <value type="bool" key="RunConfiguration.UseCppDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
    <value type="bool" key="RunConfiguration.UseMultiProcess">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
    <value type="QString" key="RunConfiguration.WorkingDirectory"></value>
    <value type="QString" key="RunConfiguration.WorkingDirectory.default"></value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.Target.3</variable>
  <valuemap type="QVariantMap">
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Desktop Qt 5.14.2 MinGW 64-bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Desktop Qt 5.14.2 MinGW 64-bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">qt.qt5.5142.win64_mingw73_kit</value>
   <value type="int" key="ProjectExplorer.Target.ActiveBuildConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.0">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MinGW_64_bit-Debug</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">false</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Debug</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">2</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.1">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MinGW_64_bit-Release</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">false</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Release</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.2">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/qtlai/build-texturesurface-Desktop_Qt_5_14_2_MinGW_64_bit-Profile</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">true</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Profile</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.BuildConfigurationCount">3</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.PluginSettings"/>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
    <value type="QString" key="Analyzer.Perf.CallgraphMode">dwarf</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.Events">
     <value type="QString">cpu-cycles</value>
    </valuelist>
    <valuelist type="QVariantList" key="Analyzer.Perf.ExtraArguments"/>
    <value type="int" key="Analyzer.Perf.Frequency">250</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.RecordArguments">
     <value type="QString">-e</value>
     <value type="QString">cpu-cycles</value>
     <value type="QString">--call-graph</value>
     <value type="QString">dwarf,4096</value>
     <value type="QString">-F</value>
     <value type="QString">250</value>
    </valuelist>
    <value type="QString" key="Analyzer.Perf.SampleMode">-F</value>
    <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
    <value type="int" key="Analyzer.Perf.StackSize">4096</value>
    <value type="bool" key="Analyzer.QmlProfiler.AggregateTraces">false</value>
    <value type="bool" key="Analyzer.QmlProfiler.FlushEnabled">false</value>
    <value type="uint" key="Analyzer.QmlProfiler.FlushInterval">1000</value>
    <value type="QString" key="Analyzer.QmlProfiler.LastTraceFile"></value>
    <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.AddedSuppressionFiles"/>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectBusEvents">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectSystime">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableBranchSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableCacheSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableEventToolTips">true</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.MinimumCostRatio">0.01</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.VisualisationMinimumCostRatio">10</value>
    <value type="bool" key="Analyzer.Valgrind.FilterExternalIssues">true</value>
    <value type="QString" key="Analyzer.Valgrind.KCachegrindExecutable">kcachegrind</value>
    <value type="int" key="Analyzer.Valgrind.LeakCheckOnFinish">1</value>
    <value type="int" key="Analyzer.Valgrind.NumCallers">25</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.RemovedSuppressionFiles"/>
    <value type="int" key="Analyzer.Valgrind.SelfModifyingCodeDetection">1</value>
    <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
    <value type="bool" key="Analyzer.Valgrind.ShowReachable">false</value>
    <value type="bool" key="Analyzer.Valgrind.TrackOrigins">true</value>
    <value type="QString" key="Analyzer.Valgrind.ValgrindExecutable">valgrind</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.VisibleErrorKinds">
     <value type="int">0</value>
     <value type="int">1</value>
     <value type="int">2</value>
     <value type="int">3</value>
     <value type="int">4</value>
     <value type="int">5</value>
     <value type="int">6</value>
     <value type="int">7</value>
     <value type="int">8</value>
     <value type="int">9</value>
     <value type="int">10</value>
     <value type="int">11</value>
     <value type="int">12</value>
     <value type="int">13</value>
     <value type="int">14</value>
    </valuelist>
    <value type="int" key="PE.EnvironmentAspect.Base">2</value>
    <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
    <value type="QString" key="ProjectExplorer.CustomExecutableRunConfiguration.Executable"></value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.CustomExecutableRunConfiguration</value>
    <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey"></value>
    <value type="QString" key="RunConfiguration.Arguments"></value>
    <value type="bool" key="RunConfiguration.Arguments.multi">false</value>
    <value type="QString" key="RunConfiguration.OverrideDebuggerStartup"></value>
    <value type="bool" key="RunConfiguration.UseCppDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
    <value type="bool" key="RunConfiguration.UseMultiProcess">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
    <value type="QString" key="RunConfiguration.WorkingDirectory"></value>
    <value type="QString" key="RunConfiguration.WorkingDirectory.default"></value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.TargetCount</variable>
  <value type="int">4</value>
 </data>
 <data>
  <variable>ProjectExplorer.Project.Updater.FileVersion</variable>
  <value type="int">22</value>
 </data>
 <data>
  <variable>Version</variable>
  <value type="int">22</value>
 </data>
</qtcreator>

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE QtCreatorProject>
<!-- Written by QtCreator 4.11.1, 2023-09-06T22:14:06. -->
<qtcreator>
 <data>
  <variable>EnvironmentId</variable>
  <value type="QByteArray">{ea4a7afc-ab49-42f7-b15b-0718868a5bcf}</value>
 </data>
 <data>
  <variable>ProjectExplorer.Project.ActiveTarget</variable>
  <value type="int">0</value>
 </data>
 <data>
  <variable>ProjectExplorer.Project.EditorSettings</variable>
  <valuemap type="QVariantMap">
   <value type="bool" key="EditorConfiguration.AutoIndent">true</value>
   <value type="bool" key="EditorConfiguration.AutoSpacesForTabs">false</value>
   <value type="bool" key="EditorConfiguration.CamelCaseNavigation">true</value>
   <valuemap type="QVariantMap" key="EditorConfiguration.CodeStyle.0">
    <value type="QString" key="language">Cpp</value>
    <valuemap type="QVariantMap" key="value">
     <value type="QByteArray" key="CurrentPreferences">CppGlobal</value>
    </valuemap>
   </valuemap>
   <valuemap type="QVariantMap" key="EditorConfiguration.CodeStyle.1">
    <value type="QString" key="language">QmlJS</value>
    <valuemap type="QVariantMap" key="value">
     <value type="QByteArray" key="CurrentPreferences">QmlJSGlobal</value>
    </valuemap>
   </valuemap>
   <value type="int" key="EditorConfiguration.CodeStyle.Count">2</value>
   <value type="QByteArray" key="EditorConfiguration.Codec">System</value>
   <value type="bool" key="EditorConfiguration.ConstrainTooltips">false</value>
   <value type="int" key="EditorConfiguration.IndentSize">4</value>
   <value type="bool" key="EditorConfiguration.KeyboardTooltips">false</value>
   <value type="int" key="EditorConfiguration.MarginColumn">80</value>
   <value type="bool" key="EditorConfiguration.MouseHiding">true</value>
   <value type="bool" key="EditorConfiguration.MouseNavigation">true</value>
   <value type="int" key="EditorConfiguration.PaddingMode">1</value>
   <value type="bool" key="EditorConfiguration.ScrollWheelZooming">true</value>
   <value type="bool" key="EditorConfiguration.ShowMargin">false</value>
   <value type="int" key="EditorConfiguration.SmartBackspaceBehavior">0</value>
   <value type="bool" key="EditorConfiguration.SmartSelectionChanging">true</value>
   <value type="bool" key="EditorConfiguration.SpacesForTabs">true</value>
   <value type="int" key="EditorConfiguration.TabKeyBehavior">0</value>
   <value type="int" key="EditorConfiguration.TabSize">8</value>
   <value type="bool" key="EditorConfiguration.UseGlobal">true</value>
   <value type="int" key="EditorConfiguration.Utf8BomBehavior">1</value>
   <value type="bool" key="EditorConfiguration.addFinalNewLine">true</value>
   <value type="bool" key="EditorConfiguration.cleanIndentation">true</value>
   <value type="bool" key="EditorConfiguration.cleanWhitespace">true</value>
   <value type="bool" key="EditorConfiguration.inEntireDocument">false</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.PluginSettings</variable>
  <valuemap type="QVariantMap"/>
 </data>
 <data>
  <variable>ProjectExplorer.Project.Target.0</variable>
  <valuemap type="QVariantMap">
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Desktop Qt 5.14.2 MSVC2017 64bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Desktop Qt 5.14.2 MSVC2017 64bit</value>
   <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">qt.qt5.5142.win64_msvc2017_64_kit</value>
   <value type="int" key="ProjectExplorer.Target.ActiveBuildConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveDeployConfiguration">0</value>
   <value type="int" key="ProjectExplorer.Target.ActiveRunConfiguration">0</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.0">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/PCLcode/texturesurface/texturesurface</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">false</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Debug</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">2</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.1">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/PCLcode/texturesurface/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_64bit-Release</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">false</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Release</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.BuildConfiguration.2">
    <value type="QString" key="ProjectExplorer.BuildConfiguration.BuildDirectory">D:/PCLcode/texturesurface/build-texturesurface-Desktop_Qt_5_14_2_MSVC2017_64bit-Profile</value>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">QtProjectManager.QMakeBuildStep</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.LinkQmlDebuggingLibrary">true</value>
      <value type="QString" key="QtProjectManager.QMakeBuildStep.QMakeArguments"></value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.QMakeForced">false</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.SeparateDebugInfo">true</value>
      <value type="bool" key="QtProjectManager.QMakeBuildStep.UseQtQuickCompiler">true</value>
     </valuemap>
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.1">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">false</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments"></value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">2</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Build</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Build</value>
    </valuemap>
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.1">
     <valuemap type="QVariantMap" key="ProjectExplorer.BuildStepList.Step.0">
      <value type="bool" key="ProjectExplorer.BuildStep.Enabled">true</value>
      <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.MakeStep</value>
      <valuelist type="QVariantList" key="Qt4ProjectManager.MakeStep.BuildTargets"/>
      <value type="bool" key="Qt4ProjectManager.MakeStep.Clean">true</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeArguments">clean</value>
      <value type="QString" key="Qt4ProjectManager.MakeStep.MakeCommand"></value>
      <value type="bool" key="Qt4ProjectManager.MakeStep.OverrideMakeflags">false</value>
     </valuemap>
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">1</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Clean</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Clean</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">2</value>
    <value type="bool" key="ProjectExplorer.BuildConfiguration.ClearSystemEnvironment">false</value>
    <valuelist type="QVariantList" key="ProjectExplorer.BuildConfiguration.UserEnvironmentChanges"/>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Profile</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">Qt4ProjectManager.Qt4BuildConfiguration</value>
    <value type="int" key="Qt4ProjectManager.Qt4BuildConfiguration.BuildConfiguration">0</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.BuildConfigurationCount">3</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.DeployConfiguration.0">
    <valuemap type="QVariantMap" key="ProjectExplorer.BuildConfiguration.BuildStepList.0">
     <value type="int" key="ProjectExplorer.BuildStepList.StepsCount">0</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DefaultDisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.DisplayName">Deploy</value>
     <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.BuildSteps.Deploy</value>
    </valuemap>
    <value type="int" key="ProjectExplorer.BuildConfiguration.BuildStepListCount">1</value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.DefaultDeployConfiguration</value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.DeployConfigurationCount">1</value>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.PluginSettings"/>
   <valuemap type="QVariantMap" key="ProjectExplorer.Target.RunConfiguration.0">
    <value type="QString" key="Analyzer.Perf.CallgraphMode">dwarf</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.Events">
     <value type="QString">cpu-cycles</value>
    </valuelist>
    <valuelist type="QVariantList" key="Analyzer.Perf.ExtraArguments"/>
    <value type="int" key="Analyzer.Perf.Frequency">250</value>
    <valuelist type="QVariantList" key="Analyzer.Perf.RecordArguments">
     <value type="QString">-e</value>
     <value type="QString">cpu-cycles</value>
     <value type="QString">--call-graph</value>
     <value type="QString">dwarf,4096</value>
     <value type="QString">-F</value>
     <value type="QString">250</value>
    </valuelist>
    <value type="QString" key="Analyzer.Perf.SampleMode">-F</value>
    <value type="bool" key="Analyzer.Perf.Settings.UseGlobalSettings">true</value>
    <value type="int" key="Analyzer.Perf.StackSize">4096</value>
    <value type="bool" key="Analyzer.QmlProfiler.AggregateTraces">false</value>
    <value type="bool" key="Analyzer.QmlProfiler.FlushEnabled">false</value>
    <value type="uint" key="Analyzer.QmlProfiler.FlushInterval">1000</value>
    <value type="QString" key="Analyzer.QmlProfiler.LastTraceFile"></value>
    <value type="bool" key="Analyzer.QmlProfiler.Settings.UseGlobalSettings">true</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.AddedSuppressionFiles"/>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectBusEvents">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.CollectSystime">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableBranchSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableCacheSim">false</value>
    <value type="bool" key="Analyzer.Valgrind.Callgrind.EnableEventToolTips">true</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.MinimumCostRatio">0.01</value>
    <value type="double" key="Analyzer.Valgrind.Callgrind.VisualisationMinimumCostRatio">10</value>
    <value type="bool" key="Analyzer.Valgrind.FilterExternalIssues">true</value>
    <value type="QString" key="Analyzer.Valgrind.KCachegrindExecutable">kcachegrind</value>
    <value type="int" key="Analyzer.Valgrind.LeakCheckOnFinish">1</value>
    <value type="int" key="Analyzer.Valgrind.NumCallers">25</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.RemovedSuppressionFiles"/>
    <value type="int" key="Analyzer.Valgrind.SelfModifyingCodeDetection">1</value>
    <value type="bool" key="Analyzer.Valgrind.Settings.UseGlobalSettings">true</value>
    <value type="bool" key="Analyzer.Valgrind.ShowReachable">false</value>
    <value type="bool" key="Analyzer.Valgrind.TrackOrigins">true</value>
    <value type="QString" key="Analyzer.Valgrind.ValgrindExecutable">valgrind</value>
    <valuelist type="QVariantList" key="Analyzer.Valgrind.VisibleErrorKinds">
     <value type="int">0</value>
     <value type="int">1</value>
     <value type="int">2</value>
     <value type="int">3</value>
     <value type="int">4</value>
     <value type="int">5</value>
     <value type="int">6</value>
     <value type="int">7</value>
     <value type="int">8</value>
     <value type="int">9</value>
     <value type="int">10</value>
     <value type="int">11</value>
     <value type="int">12</value>
     <value type="int">13</value>
     <value type="int">14</value>
    </valuelist>
    <value type="int" key="PE.EnvironmentAspect.Base">2</value>
    <valuelist type="QVariantList" key="PE.EnvironmentAspect.Changes"/>
    <value type="QString" key="ProjectExplorer.CustomExecutableRunConfiguration.Executable"></value>
    <value type="QString" key="ProjectExplorer.ProjectConfiguration.Id">ProjectExplorer.CustomExecutableRunConfiguration</value>
    <value type="QString" key="ProjectExplorer.RunConfiguration.BuildKey"></value>
    <value type="QString" key="RunConfiguration.Arguments"></value>
    <value type="bool" key="RunConfiguration.Arguments.multi">false</value>
    <value type="QString" key="RunConfiguration.OverrideDebuggerStartup"></value>
    <value type="bool" key="RunConfiguration.UseCppDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseCppDebuggerAuto">true</value>
    <value type="bool" key="RunConfiguration.UseMultiProcess">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebugger">false</value>
    <value type="bool" key="RunConfiguration.UseQmlDebuggerAuto">true</value>
    <value type="QString" key="RunConfiguration.WorkingDirectory"></value>
    <value type="QString" key="RunConfiguration.WorkingDirectory.default"></value>
   </valuemap>
   <value type="int" key="ProjectExplorer.Target.RunConfigurationCount">1</value>
  </valuemap>
 </data>
 <data>
  <variable>ProjectExplorer.Project.TargetCount</variable>
  <value type="int">1</value>
 </data>
 <data>
  <variable>ProjectExplorer.Project.Updater.FileVersion</variable>
  <value type="int">22</value>
 </data>
 <data>
  <variable>Version</variable>
  <value type="int">22</value>
 </data>
</qtcreator>

/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#include "topographicseries.h"

//! [0]
// Value used to encode height data as RGB value on PNG file
const float packingFactor = 11983.0f;
//! [0]

TopographicSeries::TopographicSeries()
{
    setDrawMode(QSurface3DSeries::DrawSurface);
    setFlatShadingEnabled(true);
    setBaseColor(Qt::white);
}

TopographicSeries::~TopographicSeries()
{
}

void TopographicSeries::setTopographyFile(const QString file, float width, float height)
//! [1]
{
    QImage heightMapImage(file);
    uchar *bits = heightMapImage.bits();
    int imageHeight = heightMapImage.height();
    int imageWidth = heightMapImage.width();
    int widthBits = imageWidth * 4;
    float stepX = width / float(imageWidth);
    float stepZ = height / float(imageHeight);

    QSurfaceDataArray *dataArray = new QSurfaceDataArray;
    dataArray->reserve(imageHeight);
    for (int i = 0; i < imageHeight; i++) {
        int p = i * widthBits;
        float z = height - float(i) * stepZ;
        QSurfaceDataRow *newRow = new QSurfaceDataRow(imageWidth);
        for (int j = 0; j < imageWidth; j++) {
            uchar aa = bits[p + 0];
            uchar rr = bits[p + 1];
            uchar gg = bits[p + 2];
            uint color = uint((gg << 16) + (rr << 8) + aa);
            float y = float(color) / packingFactor;
            (*newRow)[j].setPosition(QVector3D(float(j) * stepX, y, z));
            p = p + 4;
        }
        *dataArray << newRow;
    }

    dataProxy()->resetArray(dataArray);
//! [1]

    m_sampleCountX = float(imageWidth);
    m_sampleCountZ = float(imageHeight);
}

/****************************************************************************
**
** Copyright (C) 2016 The Qt Company Ltd.
** Contact: https://www.qt.io/licensing/
**
** This file is part of the Qt Data Visualization module of the Qt Toolkit.
**
** $QT_BEGIN_LICENSE:GPL$
** Commercial License Usage
** Licensees holding valid commercial Qt licenses may use this file in
** accordance with the commercial license agreement provided with the
** Software or, alternatively, in accordance with the terms contained in
** a written agreement between you and The Qt Company. For licensing terms
** and conditions see https://www.qt.io/terms-conditions. For further
** information use the contact form at https://www.qt.io/contact-us.
**
** GNU General Public License Usage
** Alternatively, this file may be used under the terms of the GNU
** General Public License version 3 or (at your option) any later version
** approved by the KDE Free Qt Foundation. The licenses are as published by
** the Free Software Foundation and appearing in the file LICENSE.GPL3
** included in the packaging of this file. Please review the following
** information to ensure the GNU General Public License requirements will
** be met: https://www.gnu.org/licenses/gpl-3.0.html.
**
** $QT_END_LICENSE$
**
****************************************************************************/

#ifndef TOPOGRAPHICSERIES_H
#define TOPOGRAPHICSERIES_H

#include <QtDataVisualization/QSurface3DSeries>

class TopographicSeries : public QSurface3DSeries
{
    Q_OBJECT
public:
    explicit TopographicSeries();
    ~TopographicSeries();

    void setTopographyFile(const QString file, float width, float height);

    float sampleCountX() { return m_sampleCountX; }
    float sampleCountZ() { return m_sampleCountZ; }

public Q_SLOTS:

private:
    float m_sampleCountX;
    float m_sampleCountZ;
};

#endif // TOPOGRAPHICSERIES_H
